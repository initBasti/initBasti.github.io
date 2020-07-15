# Hotplug automation

In this post, I will explain what happens within your linux machine when you plug in a USB keyboard. A quick dive into the internal workings of the host controller and usb driver will lead into the functionality of the udev program. And at the end, I'll try to explain how to implement automatic module loading within your module. 

If you are interested in the journey so far, checkout:  
[1. Hello world module](https://sebastianfricke.me/eudyptula-challenge-Part1/)  
[2. Building the kernel](https://sebastianfricke.me/eudyptula-challenge-Part-2/)  
[3. Modify & Patch the kernel](https://sebastianfricke.me/eudyptula-challenge-Part-3/)  
[4. Coding style practice](https://sebastianfricke.me/eudyptula-challenge-Part-4/)  

## Task
Yeah, you survived the coding style mess! Now, on to some "real" things, as I know you are getting bored by these so far.

So, two simple tasks this time around:

Take the kernel module you wrote for task 01, and modify it so that when any USB keyboard is plugged in, the module will be automatically loaded by the correct userspace hotplug tools (which are implemented by depmod / kmod / udev / mdev / systemd, depending on what distro you are using.)
Again, provide "proof" this all works.

Yes, so simple, and yet, it's a bit tricky. As a hint, go read chapter 14 of the book, "Linux Device Drivers, 3rd edition." Don't worry, it's free, and online, no need to go buy anything.

## Preparation
I started by working myself through chapter 14 of the LDD book and I learned a lot. But I quickly discovered that I was running into some dead ends. Especially, the "Hotplug Event Generation" part contained elements, which I simply could not locate on my system like: `/sbin/hotplug`.

In order to find out, what happens inside the kernel whenever a USB device is connected to/disconnected from the bus. I will investigate the following steps:  
1. What happens after a USB device is plugged in?
2. What are the different possibilities of loading a module on a certain event?

___

### 1. What happens after a USB device is plugged in?

*prolog*  

In this part, I will explain how a computer system reacts to a newly inserted device. This explanation goes above and beyond the actual task, which is more targeted on using the correct kernel tools for the job. If you are only interested in using the different tools properly, you can skip ahead to the execution and proof parts.

*overview*  

Generally speaking there are four major actors, within this chain of events: the **host controller interface**, **host controller driver**, **USB driver**, and **udev**.  
The host controller interface detects a new device and triggers an interrupt, which wakes up a process to handle USB hub events. Within this process the system determines where the event happened and the type of event. The port is prepared for the device by enabling it and initializing the port data structures, like the 'Slot context' that contains information about the device and the 'Input context', which specifies endpoints.  The process continues by attaching an address to the device,  configuring & registering the device, and assigning the proper driver. User-space is notified, through the creation of sysfs entries and sending device/event information to the Udev daemon via a Netlink socket.

___

*detailed view*  

#### Terminology:  

##### What is a host controller?  
The host controller is a piece of hardware installed to the motherboard, that **manages the communication of multiple devices** connected to the USB root hubs.
It creates an efficient data transfer and acts as a layer between the USB driver and the USB device. There are different host controller interfaces (OHCI, UHCI, EHCI, XHCI) that enable the host controller to communicate with the host controller driver.


*How does the host controller manage multiple USB devices?*
- device slots describe the data structures of a USB device
- Each device is represented by an entry in the Device Context Base Address Array (contains pointers to the base address), a register in the Doorbell Array register (used by software to notify the host controller about pending work), and a device’s Device Context (with information about the device and it's endpoints).
- It contains 3 rings for communication:
    + Command ring
        * Pass commands to the host controller (read-only for the host controller)
    + Event ring
        * command completion and async events to OS (read-only for the OS)
    + Transfer ring
        * circular queue of transfer descriptors for a single endpoint
        * transfer descriptors contain data buffers to/from the USB
        * read-only for the host controller

##### What is a USB root hub?  
The purpose of a USB root hub is to connect several USB devices to the system host. A typical system often contains different USB root hubs for multiple USB specifications (2.0, 3.0, etc).
It can be viewed as a tree of USB devices, that is managed by the hub driver and connected to the host controller.

##### And what is a 'sysfs'?  

A representation of the Linux device model as a pseudo filesystem, that allows easy export of information about subsystems, device drivers, and their associated hardware counterparts. The Linux device model is a high-level functional view upon a system, that describes the different parts in an object-oriented manner. The entries within this filesystem are the basis upon which **udev** works.


#### The sequence of actions

When the USB root hub is initialized the `hub_probe` function is registered to the driver as it's probe function.
Whenever it is triggered, the USB hub is set up and a [kworker](https://askubuntu.com/a/52299/839014) is created that processes events with: `INIT_WORK(&hub->events, hub_event);`.
Whenever the root hub port detects a new device connection the `hub_irq` function is activated, which wakes up the kworker that is connected to the `hub_event` function.

Now to explain the steps, that follow take a look at the steps described in the [XHCI specification](https://www.intel.com/content/www/us/en/products/docs/io/universal-serial-bus/extensible-host-controler-interface-usb-xhci.html) on page 83-85. The connection change is handled by the host controller, by setting the CSC flag to 1 and posting a port status change event to the event ring, the system has to identify on which port an event took place. You can watch that step within the `hub_event` function:  

    for (i = 1; i <= hdev->maxchild; i++) {
        struct usb_port *port_dev = hub->ports[i - 1];

        if (test_bit(i, hub->event_bits)
                || test_bit(i, hub->change_bits)
                || test_bit(i, hub->wakeup_bits)) {
            ...
            port_event(hub, i);
            ...
        }
    }

Now, that we found the correct port, it is important to know what kind of action is required (insert/detach/reset). A call to `hub_port_status` reads the bits of the `PORTSC` register to get that piece of information.  

    static int hub_ext_port_status(struct usb_hub *hub, int port1, int type, u16 *status, u16 *change, u32 *ext_status)
    {
        ...
        ret = get_port_status(hub->hdev, port1, &hub->status->port, type, len);
        ...
            *status = le16_to_cpu(hub->status->port.wPortStatus);
            *change = le16_to_cpu(hub->status->port.wPortChange);
        ...
    }

Now, that we found the correct port, it is important to know what kind of action is required (insert/detach/reset). A call to `hub_port_status` reads the bits of the `PORTSC` register to get that piece of information.  

On a connection change, the function `hub_port_connect_change` checks if it can restore an existing device before calling `hub_port_connect`.

Which is responsible for setting up the USB device on the hub.  
Configure and allocate the device:


    static void hub_port_connect(struct usb_hub *hub, int port1, u16 portstatus, u16 portchange)
    {
        ...
        udev = usb_alloc_dev(hdev, hdev->bus, port1);
        ...

Enable & address the device and get a device descriptor within:  
    ...
    status = hub_port_init(hub, udev, port1, i);
    ...
Register the device and find a driver:  
        ...
        status = usb_new_device(udev);
        ...
    }

Within `usb_alloc_dev` a device slot is acquired by calling the host controller function `xhci_alloc_dev`:
    int xhci_alloc_dev(struct usb_hcd *hcd, struct usb_device *udev)
    {
        ...
        ret = xhci_queue_slot_control(xhci, command, TRB_ENABLE_SLOT, 0);
        ...
    }

A device slot describes the generic set of data structures used by a USB device through the XHCI interface.  
Those data structures are for example:  


**Input context**:
* Which contains the input control context
    + defines which device context data structures are affected by commands
* And also contains the **device context**, which splits up into slot and endpoint context data structures
    + **Slot context**:
        - define the information that applies to whole devices
        - like the USB address, state, parent port number, speed and number of ports
    + **Endpoint context**:
        - defines information about a specific endpoint
        - like the endpoint state or transfer package attributes


In `hub_port_init`, the hub driver makes calls to the host controller driver to enable the device and get an address for the device:  
    static int
    hub_port_init(struct usb_hub *hub, struct usb_device *udev, int port1,
            int retry_counter)
    {
        ...
        for (retries = 0; retries < GET_DESCRIPTOR_TRIES; (++retries, msleep(100))) {
            ...
                retval = hub_enable_device(udev);
                ...

                    retval = hub_set_address(udev, devnum);
            ...
    }

When the device is addressed, the default control endpoint (ep0) is activated, which enables the system to get access to the device's configuration, status, and control information.

Within `usb_new_device`, a call to `device_add` includes the device into `sysfs` and finally executes `kobject_uevent`:  
    int device_add(struct device *dev)
    {
        ...
        error = kobject_add(&dev->kobj, dev->kobj.parent, NULL);
        ...
        kobject_uevent(&dev->kobj, KOBJ_ADD);
        ...
    }

The uevent function writes arguments, that specify the device and the event, directly to a Netlink socket. A Netlink socket is a special tool for asynchronous inter-process communication of the kernel and userspace. Those strings are then read by the udev daemon, which lurks on the socket for messages and splits them into their sub-components. The user can specify rules, that trigger if arguments match a rule.

*udev(userspace) part*  

Alright, the udev daemon just read new input from the Netlink socket, what now?

Type `sudo udevadm monitor` and proceed to unplug & plug in your keyboard. You will get a bunch of output that looks similar to this:  

```
KERNEL[3790.687404] add      /devices/pci0000:00/0000:00:14.0/usb1/1-1/1-1.4 (usb)
KERNEL[3790.695192] add      /devices/pci0000:00/0000:00:14.0/usb1/1-1/1-1.4/1-1.4:1.0 (usb)
KERNEL[3790.702318] add      /devices/pci0000:00/0000:00:14.0/usb1/1-1/1-1.4/1-1.4:1.0/0003:045E:00DB.000E (hid)
KERNEL[3790.702459] add      /devices/pci0000:00/0000:00:14.0/usb1/1-1/1-1.4/wakeup/wakeup29 (wakeup)
KERNEL[3790.702527] add      /devices/pci0000:00/0000:00:14.0/usb1/1-1/1-1.4/1-1.4:1.0/0003:045E:00DB.000E/input/input27 (input)
....
KERNEL[3790.762076] add      /devices/pci0000:00/0000:00:14.0/usb1/1-1/1-1.4/1-1.4:1.0/0003:045E:00DB.000E/hidraw/hidraw3 (hidraw)
KERNEL[3790.762119] bind     /devices/pci0000:00/0000:00:14.0/usb1/1-1/1-1.4/1-1.4:1.0/0003:045E:00DB.000E (hid)
KERNEL[3790.762162] bind     /devices/pci0000:00/0000:00:14.0/usb1/1-1/1-1.4/1-1.4:1.0 (usb)
KERNEL[3790.762204] add      /devices/pci0000:00/0000:00:14.0/usb1/1-1/1-1.4/1-1.4:1.1 (usb)
KERNEL[3790.772905] add      /devices/pci0000:00/0000:00:14.0/usb1/1-1/1-1.4/1-1.4:1.1/0003:045E:00DB.000F (hid)
KERNEL[3790.773342] add      /devices/pci0000:00/0000:00:14.0/usb1/1-1/1-1.4/1-1.4:1.1/0003:045E:00DB.000F/input/input28 (input)
KERNEL[3790.829925] add      /devices/pci0000:00/0000:00:14.0/usb1/1-1/1-1.4/1-1.4:1.1/0003:045E:00DB.000F/input/input28/event18 (input)
KERNEL[3790.830043] add      /devices/pci0000:00/0000:00:14.0/usb1/1-1/1-1.4/1-1.4:1.1/0003:045E:00DB.000F/hidraw/hidraw4 (hidraw)
KERNEL[3790.830073] bind     /devices/pci0000:00/0000:00:14.0/usb1/1-1/1-1.4/1-1.4:1.1/0003:045E:00DB.000F (hid)
KERNEL[3790.830098] bind     /devices/pci0000:00/0000:00:14.0/usb1/1-1/1-1.4/1-1.4:1.1 (usb)
KERNEL[3790.830124] bind     /devices/pci0000:00/0000:00:14.0/usb1/1-1/1-1.4 (usb)
```
This is the information, that is received and processed by the [udev daemon](https://www.freedesktop.org/software/systemd/man/systemd-udevd.service.html). As we can see above, the kernel is creating a multitude of directories in the `/sys/` directory (sysfs). And we can also spot the subsystems related with the keyboard (usb, hid, input).  

Next I'm going to show how we handle those information and what we can do with them.  

### 2. What are the different possibilities of loading a module on a certain event?

It is now clear, that the udev daemon receives information about the device, which can be used to match to a specific event caused by a device. So in order to load the module, when the keyboard is plugged in, we require a tool that can interact with that information. Here are the two best ways, that do the job:  
* creating a udev rule in `/etc/udev/rules.d/`
* using the MODULE_DEVICE_TABLE macro within the module.

Additionally, there is also the possibility of using the modprobe.d configuration file, in order to create a soft dependency between your module and a driver. But I leave this out here, as this interacts with inserted modules instead of inserted devices and if the specific module is already loaded, my module would not be triggered on a new insertion of a device.

### 2.1 How to use udev

A lot has changed within these parts of the system, for a detailed list of changes look at: the [Suse_linux manual](https://www.pks.mpg.de/~mueller/docs/suse10.1/suselinux-manual_en/manual/sec.udev.hplug.html). I had to make some research in order to get a up-to-date view on the information provided by the [book](https://static.lwn.net/images/pdf/LDD3/ch14.pdf).
First, udev no longer relies on `/sbin/hotplug`, to be more precise `/sbin/hotplug` is no longer actively used except for the short time window before rootfs is running. And a [netlink socket](https://man7.org/linux/man-pages/man7/netlink.7.html) is now used for the communication between the kernel and the udev daemon.

What can we do with udev?
* **Get information about recent events:**  
The `udevadm monitor` command provides a live view of recent events, you can filter events by providing a subsystem or reduce the events to kernel/udev only. More information can be found with `udevadm monitor -h`.
* **Find information about a specific device:**  
With `udevadm info` you are able to search for more specific information about either a sysfs entry (for example with a result of `udevadm monitor`) or a entry in the `/dev` directory by using the `--name` option.
* **Find inherited attributes by the parents:**  
You can even investigate the relationship of the device to other subsystems. Just type `udevadm info {DEVICE} --attribute-walk` and find the appropriate inherited attribute of the underlying device.
* **Create rules to trigger specific activities on various actions:**  
Within `/etc/udev/rules.d/` you have the ability to create custom `*.rules` files, to trigger specific actions on specified events. Here a few examples:  

  ---
  + *Create a folder, whenever a specific device is connected and remove it when the device is disconnected*:  

    **udev rule**(within `/etc/udev/rules.d/*.rules` for debian):  
    ```
    ACTION=="add"\
    , ATTRS{serial}=="xxxxxxxxxxxx"\
    , ATTRS{idVendor}=="xxxx"\
    , RUN+="/usr/bin/mkdir -p /media/usb_storage"

    ACTION=="remove"\
    , ATTRS{serial}=="xxxxxxxxxx"\
    , ATTRS{idVendor}=="xxxx"\
    , RUN+="/usr/bin/rmdir /media/usb_storage"
    ```

  ---
  + *Play a audio file, when a device is connected*:  

    **udev rule**(within `/etc/udev/rules.d/*.rules` for debian):  
    ```
    ACTION=="add"\
    , KERNEL=="sd?[0-9]"\
    , ATTRS{product}=="USB DISK 2.0"\
    , TAG+="systemd"\
    , ENV{SYSTEMD_WANTS}="playaudio.service"
    ```

    **sytemd service**(within `/etc/systemd/system/*.service` for debian):  
    ```
    [Unit]
    Description="Play a audio file when the USB stick is connected"

    [Service]
    Type=simple
    ExecStart=/usr/bin/play -q /home/basti/Music/deep-pipe.wav
    ```

    *In my tests, I was not able to execute the play command directly from a udev rule, so I used a systemd service. This method has an additional advantage as it provides a command logging feature. (systemctl status playaudio.service)*  

    ---

  + *Automatically mount a external hard-drive*:  

    **udev rule**(within `/etc/udev/rules.d/*.rules` for debian):  
    ```
    ACTION=="add"\
    , KERNEL=="sd?[0-9]"\
    , ATTRS{product}=="USB DISK 2.0"\
    , TAG+="systemd"\
    , ENV{SYSTEMD_WANTS}="automount.service"\
    , SYMLINK+="usb_storage_x"
    ```

    **sytemd service**(within `/etc/systemd/system/*.service` for debian):  
    ```
    [Unit]
    Description="Automount x USB stick"

    [Service]
    Type=simple
    ExecStart=/usr/bin/mount /dev/usb_storage_x /media/usb_storage
    ```

    ---

And so on... the possiblities are endless, but always be careful about possible matching errors, you don't want to accidentally dump an automatic backup onto your co-workers USB-Stick ;).


What can go wrong?
* The matching rule is not specific enough and triggered by the wrong device.  
    Solution: Use a identifier that is unique for your device like the serial number or the model name.

* The matching rule is incorrect, but you don't see why.  
    Solution: Check what happens when the rule is called with `udevadm test`.

* The priority of the rule is not high enough, so other rules take precedence.  
    Solution: If another rule takes precedence, you can increase the priority of your rule by incrementing the number in front of the rule file name (Example: `90-local.rules`).

For all of those cases the `udevadm test {sysfs entry of the device}` command can help you track down the problem. You can observe, which rules are triggered by the specific device, what commands are called and the order of actions. Additionally, as udev and systemd are pretty closely related nowadays, I recommend the use of systemd services as those provide logs and a status.

## Execution

Alright, now that all the background information is explained, how do I actually implement an automatically loaded module, without using any udev rule?  
The answer to that problem is explained on page 403 of chapter 14 in LDD3:  
> when a driver uses the MODULE_DEVICE_TABLE macro ,the program `depmod` ,takes that information and  
> creates the files located in/lib/module/KERNEL_VERSION/modules.\*map.  The \*is  different  ,  
> depending  on  the  bus  typethat the driver supports.  

Sadly, this information still requires a little bit more investigation and it also contains a *deprecated* part. On modern systems there are no `.\*map` files, instead these entries can all be found within the modules.alias file. [\[*\]](https://stackoverflow.com/a/25644147/9918329)

In order to find out how to use the `MODULE_DEVICE_TABLE` macro for the module to be loaded, whenever a usb keyboard is inserted. I first investigate an existing example in the kernel source tree.

#### Deconstruct an existing example:  

[drivers/usb/serial/navman.c](https://github.com/torvalds/linux/blob/master/drivers/usb/serial/navman.c):

```
static const struct usb_device_id id_table[] = {
	{ USB_DEVICE(0x0a99, 0x0001) },	/* Talon Technology device */
	{ USB_DEVICE(0x0df7, 0x0900) },	/* Mobile Action i-gotU */
	{ },
};
MODULE_DEVICE_TABLE(usb, id_table);
```

This contains 3 parts of interest: `struct usb_device_id`, `USB_DEVICE()` and `MODULE_DEVICE_TABLE(usb,..)`.

###### struct usb_device_id:  

This is a specific struct used to identify devices for hotplugging and probing.
As we can spot in the struct below there are three major ways of identifying a device:
+ by the device class
+ by the interface class
+ by the vendor interface

```
struct usb_device_id {
    ...
	__u16		match_flags;
    ...
	__u16		idVendor;
	__u16		idProduct;
	__u16		bcdDevice_lo;
	__u16		bcdDevice_hi;

	/* Used for device class matches */
	__u8		bDeviceClass;
	__u8		bDeviceSubClass;
	__u8		bDeviceProtocol;

	/* Used for interface class matches */
	__u8		bInterfaceClass;
	__u8		bInterfaceSubClass;
	__u8		bInterfaceProtocol;

	/* Used for vendor-specific interface matches */
	__u8		bInterfaceNumber;
    ...
};
```

And this is where the macros come into place, in order to simplify the process of creating a match table.  

###### USB_DEVICE macro:  

For example this macro:  

```
#define USB_DEVICE(vend, prod) \
	.match_flags = USB_DEVICE_ID_MATCH_DEVICE, \
	.idVendor = (vend), \
	.idProduct = (prod)
```

It uses the product ID and the vendor ID to match to a device, together with a specific match flag.  
That flag is used to specify, which fields of the struct are used for matching with a detected device.  

So going back to the example, we now know that:

```
	{ USB_DEVICE(0x0a99, 0x0001) },	/* Talon Technology device */
                    ^       ^
            This is the     |
            Vendor ID       |
                        This is the product ID
                        of the device
```

And the device is detected exactly by those 2 fields through the usage of the match flag.

###### MODULE_DEVICE_TABLE macro:  

Looking at the source code from: [include/linux/module.h](https://github.com/torvalds/linux/blob/master/include/linux/module.h), we can spot that this macro creates and alias for the ID table.
```
/* Creates an alias so file2alias.c can find device table. */
#define MODULE_DEVICE_TABLE(type, name)					\
extern typeof(name) __mod_##type##__##name##_device_table		\
  __attribute__ ((unused, alias(__stringify(name))))
```

It points to the `file2alias.c` file, which is part of the **modpost** utility. It basically creates aliases for modules to devices and places them into the `modules.alias` file.

---

Alright, so I need to create a `id_table` to match to a specific device or in our case every USB keyboard.

Where do I find the correct identifiers?  

I know from the task description, that I need a generic match to any USB keyboard. Which tells me that I have to look for the USB or the HID subsystem.  
We can find examples of those match identifiers inside of the generic drivers for device types like: usbhid (usb human interface devices) or usbkbd (usb keyboards).  

One example is located within [drivers/hid/usbhid/hid-core.c](https://github.com/torvalds/linux/blob/master/drivers/hid/usbhid/hid-core.c):  

```
static const struct usb_device_id hid_usb_ids[] = {
	{ .match_flags = USB_DEVICE_ID_MATCH_INT_CLASS,
		.bInterfaceClass = USB_INTERFACE_CLASS_HID },
	{ }						/* Terminating entry */
};
```

This code uses the `USB_DEVICE_ID_MATCH_INT_CLASS` to use the interface class for matching and specifies `USB_INTERFACE_CLASS_HID` to match with any device, that is part of the USB HID class.  

---

Making the module work is now the easy part, all we need to do is adding the following lines to our module:  

```
...
#include <linux/kernel.h>
#include <linux/usb.h>
#include <linux/hid.h>
...
static const struct usb_device_id id_table[] = {
    { .match_flags = USB_DEVICE_ID_MATCH_INT_CLASS,
        .bInterfaceClass = USB_INTERFACE_CLASS_HID },
    {  }
};

MODULE_DEVICE_TABLE(usb, id_table);
```

Now build the module (`make`), copy it into the modules folder (```sudo cp hello-world.ko /lib/modules/`uname -r` ```), update the dependencies (`sudo depmod -a`) and install the module (`sudo modprobe hello_world`). This step updates the dependencies within ```/lib/modules/`uname -r`/modules.dep```


## Proof

How can I find out if the module is loaded?  

Through the same methods as described within my previous post [Hello world module](https://sebastianfricke.me/eudyptula-challenge-Part1/) at the "The Test" part.  
Which breaks down to looking at the kernel ring buffer directly with `tail`, `cat` etc. or by executing `sudo dmesg`. 

The most convinient method to me is using the live-view mode of `tail`, together with an Identifier within a `printk`.

code:
```
printk(KERN_INFO "HEWO: Hello world!\n");
```

command:  
`sudo tail -f /var/log/kern.log | grep 'HEWO'`

## Reference
[\[1. Answer on askubuntu explaining what a kworker is\]](https://askubuntu.com/a/52299/839014)  
[\[2. XHCI specification by Intel\]](https://www.intel.com/content/www/us/en/products/docs/io/universal-serial-bus/extensible-host-controler-interface-usb-xhci.html)  
[\[3. Information about the udev daemon\]](https://www.freedesktop.org/software/systemd/man/systemd-udevd.service.html)  
[\[4. Entry from the Suse linux manual about hotplugging\]](https://www.pks.mpg.de/~mueller/docs/suse10.1/suselinux-manual_en/manual/sec.udev.hplug.html)  
[\[5. Linux Device Driver book chapter 14\]](https://static.lwn.net/images/pdf/LDD3/ch14.pdf)  
[\[6. Information about the netlink socket\]](https://man7.org/linux/man-pages/man7/netlink.7.html)  
[\[7. Answer on stackoverflow that addresses the deprecated modules.\*map files\]](https://stackoverflow.com/a/25644147/9918329)  
