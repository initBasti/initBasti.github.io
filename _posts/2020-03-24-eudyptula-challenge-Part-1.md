# Printing to the kernel log

A project that was started by folks on the Linux kernel team and canceled some time ago.
When I looked at the project, I noticed that it looks like a fun way to learn how to develop for the Linux kernel.
These challenges start with the basics like printing the output to the kernel log and building the kernel.
Moving on to more demanding tasks like creating drivers, system calls, and creating real patches for the kernel that are accepted by the community.

**If you have not solved the challenge, stop right after the description of the task and try it yourself first!**

---

## Task:
Write a Linux kernel module, and stand-alone Makefile, that when loaded  
prints to the kernel debug log level, "Hello World!"  Be sure to make  
the module be able to be unloaded as well.  

The Makefile should build the kernel module against the source for the  
currently running kernel, or, use an environment variable to specify  
what kernel tree to build it against.  

Please show proof of this module being built, and running, in your  
kernel.  What this proof is is up to you, I'm sure you can come up with  
something.  Also be sure to send the kernel module you wrote, along with  
the Makefile you created to build the module.  

### Requirements:

**Debian & Ubuntu**  
`sudo apt-get install build-essentials linux-headers-$(uname -r)`  
**Fedora**  
`sudo dnf install make gcc kernel-devel kernel-headers`  

---

### What is a linux kernel module?:

Code that can be loaded or unloaded, by the kernel on demand.  
Extends the functionality of the kernel without the need to recompile the kernel.  

You can check the modules, which are currently loaded with: **lsmod**  
To get information about the module use: **modinfo \<module-name\>**  
If you want systemd to load the module on boot:  
* create a configuration file under /etc/modules-load.d/\<module-name\>.conf
* enter module names separated by new lines.

---
## Execution:

#### hello-world.c
We will work with 3 different linux kernel headers:  
  

**#include <linux/init.h>**   
*For the \_\_init and \_\_exit macros within the function definitions*  
*Used to hint to the kernel that these functions are only required*  
*for the initialization and free up phase*  

**#include <linux/module.h>**   
*For module description macros & module_init + module_exit*  

**#include <linux/kernel.h>**   
*For the printk function*  


Here is the function that is called, when the module is initialized into the kernel.  

~~~~~ c
static int __init hello_init(void)  
{  
    printk(KERN_INFO "HEWO: Hello World\n");  
    return 0;  
}
~~~~~


And here the function, for the removal of the module:

~~~~~ c
static void __exit hello_exit(void)
{
	printk(KERN_INFO "HEWO: Goodbye world\n");
}
~~~~~~
Notice here the KERN_INFO, this represents the loglevel of the message.
The Kernel provides 8 loglevels
* KERN_EMERG - Emergency condition, system is probably dead
* KERN_ALERT - Some problem has occurred, immediate attention is needed
* KERN_CRIT - A critical condition
* KERN_ERR - An error has occurred
* KERN_WARNING - A warning
* KERN_NOTICE - Normal message to take note of
* KERN_INFO - Some information
* KERN_DEBUG - Debug information related to the program

We assign these both as the functions for initialization and removal with:

~~~~~~ c
module_init(hello_init);
module_exit(hello_exit);
~~~~~~

And finally we need to write some description and license information.

~~~~~~ c
MODULE_LICENSE("GPL");
MODULE_AUTHOR("<Your Name>");
MODULE_DESCRIPTION("<What does the module do>");
MODULE_VERSION("0.1");
~~~~~~

**That's it!**
Now we need to write a makefile to compile it.

#### Makefile

This is pretty straightforward, call the Makefile within the  
/lib/modules/\<your-kernel-version\>/build folder with the parameter:  
M=$(PWD)
> The M= option causes that makefile to move back  
> into your module source directory before trying  
> to build the modules target.  
> **Linux Device Drivers, 3rd Edition, Jonathan Corbet** [\[1\]](https://www.oreilly.com/library/view/linux-device-drivers/0596005903/ch02.html)  

$(PWD) is current working directory

And the Makefile rules *build* or *clean*

~~~~~~ c
obj-m+=hello-world.o

all:
	make -C /lib/modules/$(shell uname -r)/build/ M=$(PWD) modules
clean:
	make -C /lib/modules/$(shell uname -r)/build/ M=$(PWD) clean
~~~~~~

---

## Test

You can insert a module into the kernel by using the insmod command  
together with super user do(sudo) or as root(to log to your root type: sudo su):  
`sudo insmod /location/of/file/hello-world.ko`  


### 4 ways:

**1. Use dmesg:** 

`sudo dmesg | grep world`  

**2. Watch the system log, which handles the kern.log files from the kernel ring buffer:**

`sudo cat /var/log/syslog | grep world`

*What is the kernel ring buffer?*  
A ring buffer is a implementation of a queue, as a fixed length array with
two pointers:  
one to the head and one to the tail  
Elements are added to the tail and removed from the head.  
The kernel ring buffer keeps all the log entries of the kernel.  

**3. Watch the message log:**  
*location where the syslog daemon(in my case rsyslogd) copies printk messages*
*from the kernel ring buffer*

`sudo cat /var/log/messages | grep world`

**4. Live check on second terminal window:**  

`sudo tail -f /var/log/messages | grep world`  

*To stop the live view of logging, type:*  
*Strg + C*

Unload the module with:  
`sudo rmmode hello-world`

### possible problems:
**I can only see Hello world but not Goodbye world?**
* you probably forgot the '\n' within the printk's

[\[1\]](https://www.oreilly.com/library/view/linux-device-drivers/0596005903/ch02.html) *Explanation by Jonathan Corbet about the way Makefiles work for modules*  
