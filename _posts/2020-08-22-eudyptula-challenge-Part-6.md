# Simple device driver

In this post, I'm going to explore character devices and how to interact with them.  

If you are interested in the journey so far, checkout:  
[1. Hello world module](https://sebastianfricke.me/eudyptula-challenge-Part1/)  
[2. Building the kernel](https://sebastianfricke.me/eudyptula-challenge-Part-2/)  
[3. Modify & Patch the kernel](https://sebastianfricke.me/eudyptula-challenge-Part-3/)  
[4. Coding style practice](https://sebastianfricke.me/eudyptula-challenge-Part-4/)  
[5. Exploring hotplug automation](https://sebastianfricke.me/eudyptula-challenge-Part-5/)  

**If you have not solved the challenge, stop right after the description of the task and try it yourself first!**

## Task

Nice job with the module loading macros. Those are tricky, but a very valuable skill to know about, especially when running across them in real kernel code.  

Speaking of real kernel code, let's write some!  

The tasks this time are:  

* Take the kernel module you wrote for task 01, and modify it to be a misc char device driver. The misc interface is a very simple way to be able to create a character device, without having to worry about all of the sysfs and character device registration mess. And what a mess it is, so stick to the simple interfaces wherever possible.  
* The misc device should be created with a dynamic minor number, no need running off and trying to reserve a real minor number for your test module, that would be crazy.  
* The misc device should implement the read and write functions.  
* The misc device node should show up in /dev/eudyptula.  
* When the character device node is read from, your assigned id is returned to the caller.  
* When the character device node is written to, the data sent to the kernel needs to be checked. If it matches your assigned id, then return a correct write return value. If the value does not match your assigned id, return the "invalid value" error value.  
* The misc device should be registered when your module is loaded, and unregistered when it is unloaded.  
* Provide some "proof" this all works properly.  

## Preparation

#### How does the Linux kernel handle devices?  

The Linux kernel presents a collection of data structures, which are used to represent the hardware device. This data often includes but isn't limited to a reference count, open file objects that are related to the device, the private data kept for the device, the minor and major number of the device, and a set of operations to interact with the device (which are system calls that are redirected to the device by the operating system [\[1\]](https://linux-kernel-labs.github.io/refs/heads/master/labs/device_drivers.html)).  
Every device is represented within the device model as a kobj, with all it's connections to subsystems, busses, and other devices. There are physical hardware devices and virtual 'pseudo' devices, examples for the second category are `/dev/random`, `/dev/null`, or `/dev/zero`. You can discover the devices (real & virtual), that are connected to your system within the `/dev` directory.

#### What is a character device?  

A device, that reads and writes data sequentially in the form of single characters (bytes), this includes keyboards, mouses, printers, and most 'pseudo' devices. For character devices, system calls have direct access to the device node.  

#### How do block devices differ?  

Block devices provide addressable chunks of memory, which often support random access. These devices often require more speed and also don't grant direct access to the device node but rather require an additional layer of abstraction in the form of a file-system.  

#### What is the misc device interface?  

A simplified version of a character device, which doesn't receive a major device number and only a single minor number. Used for very small devices and offers a lightweight alternative with less functionality.

#### What are major and minor numbers for devices?  

Major numbers are an offset into the Linux device driver table and describe the type of the device. Minor numbers are device instances used by the driver. Both together are used to specify, which device file points to which device.

#### What does it mean to register/unregister a device?  

When a device is registered a major and minor number gets assigned and `/dev`, `/sys` entries are created. Likewise, the unregister function deletes the devices and clears associations to the major/minor numbers.

#### How to read from a character device or write to one?  

Every character device structure contains a set of file operations (look at: [include/linux/fs.h](https://github.com/torvalds/linux/blob/master/include/linux/fs.h) for reference). If you look at the file, you can see that nearly every struct member is a function pointer, which means that you can implement functions, which are used as callback functions for the system calls. So, whenever a system call is used on the device, the system call calls the implemented function, which directs it to the device [\[2\]](http://1.droppdf.com/files/ptI6R/linux-kernel-development-3rd-edition.pdf).  
To read from  or write to a device, we can, therefore, declare functions as a callback for the read/write syscall, when we implement the device structure:  

```c
static struct file_operations dev_ops = {
    .owner = THIS_MODULE,
    .read = dev_read,
    .write = dev_write,
    .open = dev_open,
    .release = dev_release
};
```

And then we implement those functions:  

```c
static ssize_t dev_read(struct file *file_pointer, char __user *buffer,
                        size_t count, loff_t *ppos)
{
    ...
}
```

## Execution

### Module part:  

#### Defining the device structure and register the device  

Let's take a look at the `struct miscdevice` [include/linux/miscdevice.h](https://github.com/torvalds/linux/blob/master/include/linux/miscdevice.h), which will be used to create the device:  
```c
#include <linux/miscdevice.h>
#include <linux/fs.h>
#include <linux/types.h>
#include <linux/module.h>
#include <linux/kernel.h>

struct miscdevice  {
	int minor;
	const char *name;
	const struct file_operations *fops;
	struct list_head list;
	struct device *parent;
	struct device *this_device;
	const struct attribute_group **groups;
	const char *nodename;
	umode_t mode;
};
```

A lot of this structure is handled internally, all we really need to define is: `int minor`, `const char *name`.  

```c
struct miscdevice eudyptula_dev = {
	.minor = MISC_DYNAMIC_MINOR,
	.name = DEVICE_NAME,
};
```

And after this definition of the device, the remaining work for creating the `/dev` entry is trivial. Just register the device within the `__init` function and deregister it within the `__exit` function.  

```c
static int __init eudyptula_init(void)
{
	return misc_register(&eudyptula);
}

static void __exit eudyptula_exit(void)
{
	return misc_deregister(&eudyptula);
}
```

#### Setup the file operations  

Alright part one of the task is finished, if you build that module it will create the device node. But it will actually not do anything else, kinda boring..

Let's declare some operations for the driver:  

```c
static ssize_t eudyptula_read(struct file *, char __user *, size_t , loff_t *);
static ssize_t eudyptula_write(struct file *, const char __user *, size_t, loff_t *);

static struct file_operations eudyptula_fops = {
	.owner = THIS_MODULE,
	.read = eudyptula_read,
	.write = eudyptula_write,
};
```
And **add the** `.fops = &eudyptula_fops` **field to the** `struct miscdevice eudyptula`.  
The read call moves the data from the device data structure to the user and the write call moves data from the user to the device structure.  

##### What kind of data structure is needed?  

Let's say the assigned ID for the device is 50. According to the task requirements we expect the following result:
* [**READ**] `bytes_read = read(fd, buffer, n_bytes)` shall place the assigned ID of the device (eg. the minor number of the device) = "50" into the user-space buffer.
* [**WRITE**] `bytes_written = write(fd, "45", 3)`, should return: bytes_written = -1, errno = EINVAL ;  
  `bytes_written = write(fd, "50", 3)`, should return: bytes_written = 3

That means, the integer value within the minor field of the device (`eudyptula.minor`), needs to be parsed into a string and moved to the user-space buffer on a read call.
While the write call copies the user-space buffer to a kernel-space buffer before checking if the user-space buffer is equal to the kernel buffer.

So, basically all that is required is an initialized buffer with enough room to hold an ID (max. 2-5 digits probably). As well as a buffer for the content from user-space with the same size.

##### Which tools are available within the kernel for reading data from the kernel to the user/ writing data from the user to the kernel?  

`copy_to_user` and also wrappers around it like `simple_read_from_buffer`(which allows to start reading from a specified position and checks if that position is valid).  
It is important to realize, why this job cannot be done by `memcpy`. Think about it for a few seconds...

...


The important aspect to keep in mind is that the kernel can access any memory of the system, while a user-space program has a certain limited range of memory.

For more details look at this answer by the user *caf* on [stackoverflow](https://stackoverflow.com/a/12666587/9918329):


> These functions do a few things:
> * They check if the supplied userspace block is entirely within the user portion of the address space (access_ok()) - this prevents userspace applications from asking the kernel to read/write kernel addresses;
> * They return an error if any of the addresses are inaccessible, allowing the error to be returned to userspace (EFAULT) instead of crashing the kernel (this is implemented by special co-operation with the page fault handler, which specifically can detect when a fault occurs in one of the user memory access functions);
> * They allow architecture-specific magic, for example to ensure consistency on architectures with virtually-tagged caches, to disable protections like SMAP or to switch address spaces on architectures with separate user/kernel address spaces like S/390.

Writing from user-space to the kernel is performed in a similar manner, with functions like: `copy_from_user` or `simple_write_to_buffer`.  

**Note:** *You cannot use the buffer parameter (pointer to user space buffer) in* `eudyptula_write` *before*` copy_from_user` *has successfully copied the data into a kernel space buffer. This is the case because the address provided by the user space process is a locally used virtual address.*

*Is it required to protect the buffer from wrong/malicious input?*  

`copy_from_user` provides no internal buffer overflow protection, so the write operation has to check if the input is acceptable.  

For example with:  
```c
#define MAX_EUDYPTULA_BUFFER 10
...
if (count > MAX_EUDYPTULA_BUFFER) {
        return -EOVERFLOW;
}
```
#### Implementation:  

In summary, we need two buffers, one for the device's minor ID, which is parsed as a comparison string, and one as the destination buffer for copy_from_user.
And we need to implement moving data between user and kernel space while looking for overflows and invalid values.
Here is my approach:  

```c
...
#include <uapi/asm-generic/errno-base.h>
#include <linux/string.h>
...
static char intern_buffer[MAX_EUDYPTULA_BUFFER] = {0};
static char new_buffer[MAX_EUDYPTULA_BUFFER] = {0};

...

static ssize_t eudyptula_read(struct file *fp, char __user *buffer,
				    size_t count , loff_t *offset)
{
	int len = 0;
	long copy_bytes = 0;

	len = scnprintf(intern_buffer, 3, "%d", eudyptula.minor);

	copy_bytes = copy_to_user(buffer, intern_buffer, conversion_bytes);
	if (copy_bytes < 0) {
		return -EFAULT;
	}
	return copy_bytes;
}

static ssize_t eudyptula_write(struct file *fp, const char __user *buffer,
				     size_t count, loff_t *offset)
{
	int ret = 0;
	if (count > MAX_EUDYPTULA_BUFFER) {
		return -EOVERFLOW;
	}

	// Reset the buffer after each call
	if (strnlen(new_buffer, MAX_EUDYPTULA_BUFFER) > 0) {
		memset(new_buffer, 0, MAX_EUDYPTULA_BUFFER);
	}

	ret = copy_from_user(new_buffer, buffer, count);
	if (ret != 0) {
		return -EFAULT;
	}
	if (strncmp(new_buffer, intern_buffer, 4) != 0) {
		return -EINVAL;
	}
	return ret;
}
```

### User-space part:  

#### Writing the user space functions to interact with the device.  

The user-space application is really simple, the only requirments are to open the device descriptor, read from it, write to it and close it afterwards.  

Here is a quite minimal version of the read and write operations:  

```c
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <error.h>
#include <errno.h>
#include <ctype.h>

#define MSG_LEN 10

int read_from_device(char *path, char *value)
{
	int device = 0;

	if (!path || !value) {
		return -1;
	}

	device = open(path, O_RDONLY);
	if (!device) {
		perror("Device open failed");
		return -1;
	}

	if (read(device, value, strnlen(value, MSG_LEN)) < 0) {
		perror("Device read failed");
		return -1;
	}

	if (close(device) != 0) {
		printf("Warning: device node not properly closed!\n");
	}

	return 0;
}

int write_to_device(char *path, char *value)
{
	int device = 0;

	if (!path || !value) {
		return -1;
	}

	device = open(path, O_RDWR);
	if (!device) {
		perror("Device open failed");
		return -1;
	}

	if (write(device, value, strnlen(value, MSG_LEN)) < 0) {
		perror("Device write failed");
		return -1;
	}

	if (close(device) != 0) {
		printf("Warning: device node not properly closed!\n");
	}

	return 0;
}
```


#### Bonus: Creating an interface around the functions  

In order to play around with the device driver, we can create a simple CLI with the help of [getopt](https://www.gnu.org/software/libc/manual/html_node/Getopt.html).  

First, I define the possible parameter:  
```c
static struct option long_options[] = {
	{"device", required_argument, 0, 'd'},
	{"read", no_argument, 0, 'r'},
	{"write", required_argument, 0, 'w'},
	{0, 0, 0, 0}
};
```

Then I create a loop to check the arguments received through `argv`:  

```c
int main(int argc, char *argv[])
{
	int c = 0;
	int option_index = 0;
	char read_value[MSG_LEN] = {0};
	char device_node[DEV_NODE_MAX_LENGTH] = {0};
	errno = 0;

	if (argc < 2) {
		printf("Error: No device node given.\n");
		return 1;
	}
	while ((c = getopt_long(argc, argv, "d:rw:",
				long_options, &option_index)) != -1) {
		switch (c) {
		case 'd':
			if (strnlen(optarg, NODE_MAX_LENGTH) < DEV_NODE_MIN_LENGTH) {
				printf("Error: Argument too short, provide a device node.\n");
				return 1;
			}
			strncpy(device_node, optarg, DEV_NODE_MAX_LENGTH);
			if (strncmp(device_node, "/dev/", DEV_NODE_MIN_LENGTH-1) != 0) {
				printf("Error: Invalid folder, take a device from '/dev/'.\n");
				return 1;
			}
			break;
		case 'r':
			if (strnlen(device_node, DEV_NODE_MAX_LENGTH) < 1) {
				return 1;
			}
			if (read_from_device(device_node, read_value) != 0) {
				return 1;
			}
			printf("Read value: %s\n", read_value);
			break;
		case 'w':
			if (strnlen(device_node, DEV_NODE_MAX_LENGTH) < 1) {
				return 1;
			}
			if (write_to_device(device_node, optarg) != 0) {
				return 1;
			}
			printf("Device accepted write value: %s\n", optarg);
			break;
		}
	}

	return 0;
}
```

Now we can do fun stuff like:  

Read from the device:  
`sudo ./driver --device /dev/eudyptula --read'`  

Write to the device:  
`sudo ./driver -d /dev/eudyptula -w 10`  

Parse the minor ID from the read output as argument for the write call:  
`sudo ./driver -d /dev/eudyptula -w "$(sudo ./driver -d /dev/eudyptula -r | grep -Eo '[0-9]{1,3}')"`  


## Proof

### Creating simple unit tests  

In order to check and verify, that the task is completed. I create a set of unit tests for the following properties:  
* The read function should only return 0 when: The device descriptor is valid & the user has root permissions
* The write function should only return 0 when: The device descriptor is valid, the user has root permissions and the write value matches the read value
* The write function shall throw the following errors:
    + EOVERFLOW - the input given is greater than the maximum length of the module buffer
    + EINVAL - the input doesn't match the ID of the device

For the creation of the unit tests, I will use the [unity test-framework](https://www.throwtheswitch.org/unity). Which is quite easy to set up on a system:  

Download the [Zip file](https://github.com/ThrowTheSwitch/Unity/archive/master.zip) and unpack it into your project's directory.

For example:  

```
Example project  
├── kernel  
│   └── char_device_driver.c  
├── Makefile  
└── user-space  
    ├── driver.c  
    ├── operation.c  
    ├── operation.h  
    ├── test  
    │   └── unit_test.c  
    └── unity  
        └── src  
            ├── CMakeLists.txt  
            ├── meson.build  
            ├── unity.c  
            ├── unity.h  
            └── unity_internals.h  
```

The `Makefile` in this case would look like this:  

```
PATHT=user-space/test
PATHU=user-space/unity/src
PATHS=user-space
PATHK=kernel

obj-m += $(PATHK)/char_device_driver.o

all:
    make -C /lib/modules/$(shell uname -r)/build/ M=$(PWD) modules
    $(CC) $(PATHS)/driver.c $(PATHS)/operation.c -o driver
    $(CC) $(PATHU)/unity.c -o $(PATHU)/unity.o
    $(CC) $(PATHT)/unit_test.c $(PATHS)/operation.c $(PATHU)/unity.o -o unit_test.out
clean:
    make -C /lib/modules/$(shell uname -r)/build/ M=$(PWD) clean
    rm unit_test.out
    rm driver
```

The test framework is embedded within the project and compiled with a rule inside of the `Makefile`. Notice that the unit test file contains a `main` function, which means that you have to decouple the functions you want to test from the file that contains the `main` function of your application (Because a C program cannot contain two main functions).

And here is an example unit test:  

```c
#include "../unity/src/unity.h"
#include <string.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <sys/sysmacros.h>

#include "../operation.h"

#define MAX_LENGTH 40
#define WRITE_TESTS 6

void test_write_to_device(void)
{
        struct stat sb;
        char correct_value[MAX_LENGTH] = {0};

        // Initialize the result array and the expected results
        int result[WRITE_TESTS*2] = {0};
        int expected_result[WRITE_TESTS*2] = {
                -1, -1, -1, -1, -1, 0,
                -1, -1, -1, -1, -1, -1
        };

        // Setup the test cases
        char test[WRITE_TESTS][MAX_LENGTH] = {
                "-10", "0", "a", "",
                "50000000000000000000", ""
        };
        char device[2][MAX_LENGTH] = {"/dev/eudyptula", "wrong_device"};

        // Get the correct minor number of the device
        if (stat(device[0], &sb) == -1) {
                perror("lstat failed");
                return;
        }
        snprintf(correct_value, MAX_LENGTH, "%ld", (long) minor(sb.st_rdev));
        strncpy(test[WRITE_TESTS-1], correct_value, MAX_LENGTH);

        // Run the test cases
        for(int i = 0 ; i < WRITE_TESTS ; i++) {
                for(int j = 0 ; j < 2 ; j++) {
                        result[WRITE_TESTS*j + i] = write_to_device(device[j],
                                                                    test[i]);
                }
        }

        // Compare the results with the expected results
        for(int i = 0 ; i < WRITE_TESTS*2 ; i++) {
                TEST_ASSERT_EQUAL_INT(expected_result[i], result[i]);
        }
}
```

And here is the result of running the test case:  

```
$ sudo ./unit-test.out
Device write failed: Invalid argument
Device write failed: Bad file descriptor
Device write failed: Invalid argument
Device write failed: Bad file descriptor
Device write failed: Invalid argument
Device write failed: Bad file descriptor
Device write failed: Invalid argument
Device write failed: Bad file descriptor
Device write failed: Value too large for defined data type
Device write failed: Bad file descriptor
Device write failed: Bad file descriptor
Eudyptula challenge 6:88:test_write_to_device:PASS
```

The unit test checks 6 cases for 2 devices (correct & bad), which makes a total of 12 test cases. But only one test case is valid: *correct device & correct value*, so we expect 11 failing tests.  

## Reference
[1. Overview of devices in UNIX](https://linux-kernel-labs.github.io/refs/heads/master/labs/device_drivers.html)  
[2. Explanation from Robert Love about file operations in his book Linux Kernel Development](http://1.droppdf.com/files/ptI6R/linux-kernel-development-3rd-edition.pdf) on page 281  
