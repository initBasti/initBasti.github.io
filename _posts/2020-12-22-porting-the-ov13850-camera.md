# Porting the OV13850 camera to the mainline kernel

![ov13850.jpg](/images/ov13850.jpg)
[\[1\]](http://wiki.friendlyarm.com/wiki/index.php/Matrix_-_CAM1320)

## How to port a camera sensor from a downstream kernel to an upstream kernel?

This was the initial question, which bugged me after I realized, that my NanoPC-T4 was equipped with two major problems for my project.  
*Background info*:  
The main idea was to get a setup working of an SBC(Single Board Computer) with an RK3399 SOC and a Raspberry PI, in order to properly test the changes I want to produce for the libcamera. A little more background info, the Raspberry PI team has created a really nice IPA (Image Processing Algorithms) interface, which basically performs the job of getting metadata from the ISP (Image Signal Processor) feeding it into a bunch of self implemented open-source algorithms and use the output as parameters to adjust settings of the ISP. My goal is to make a lot of this possible for the rk3399 with it's rkisp1 ISP, if possible by reusing and generalizing elements of the RPi (Raspberry Pi) code.  

Now the problems that I encountered were the following:  
1. The official [kernel released by FriendlyElec](https://github.com/friendlyarm/kernel-rockchip) (the manufacturer of the NanoPC-T4), is based upon the **4.4** Kernel release, which is way too old to contain the current implementation of the RkISP1 driver and cannot be used as base for new development.
2. Additionally, the mainline kernel currently doesn't support the cameras, which are provided by FriendlyElec. Namely, the CAM1320 (OV13850) and the MCAM400 (OV4689). The MIPI-CSI-2 slot of the NanoPC-T4 has 30pins, which makes it impossible to use camera modules with 15pin FCC cables, like for example the raspberry Pi camera module with the IMX219 image sensor.

So, a solution is required, one possibillity would have been to just buy a rockPi4 (because it has a 15pin MIPI-CSI-2 slot which would fit with the IMX219 camera module). But I have already invested quite a bit into the NanoPC-T4, I want to make it work if possible. Moreover, I felt that this problem was a great opportunity to learn more about device trees. The plan is to port over the driver from the [friendlyElec Kernel](https://github.com/friendlyarm/kernel-rockchip) to the [media tree](https://git.linuxtv.org/media_tree.git/), which involves the following steps:

1. [Port the driver from 4.4 to 5.10](#driver_port)
2. [Adjust the device tree of the armbian kernel in order to set up the camera device correctly](#device_tree)
    1. [The image sensor](#image-sensor)
        1. [The clock](#image-sensor-clock)
        2. [The reset and power-down GPIOs](#image-sensor-reset-powerdown)
        3. [The pin controls](#image-sensor-pin-controls)
        4. [The port and endpoint](#image-sensor-port-endpoint)
        5. [The register](#image-sensor-register)
        6. [New changes](#dts-new)
        7. [Adjusting the device tree](#image-sensor-device-tree)
        8. [Testing the changes](#image-sensor-testing)
3. [Use the latest kernel and test the camera](#armbian_image)
4. [Create an upstream patch](#patching)

## 1. Porting the driver <a name="driver_port"></a>

In the first version of the port, I only change the things necessary for a successful build. Further, I was able to work with this driver already to a good extend, with only a few errors in libcamera and some warnings in the kernel log. The list of changes is really quite small, there was only a missing header as well as a changed function (name and parameter list) and a modified enumerator. Here is the diff:
```c
@@ -24,6 +24,7 @@
 #include <media/v4l2-ctrls.h>
 #include <media/v4l2-subdev.h>
 #include <linux/pinctrl/consumer.h>
+#include <linux/compat.h>

 #define DRIVER_VERSION                 KERNEL_VERSION(0, 0x01, 0x01)

@@ -1501,8 +1502,8 @@ static int ov13850_probe(struct i2c_client *client,
 #endif
 #if defined(CONFIG_MEDIA_CONTROLLER)
        ov13850->pad.flags = MEDIA_PAD_FL_SOURCE;
-       sd->entity.type = MEDIA_ENT_T_V4L2_SUBDEV_SENSOR;
-       ret = media_entity_init(&sd->entity, 1, &ov13850->pad, 0);
+       sd->entity.function = MEDIA_ENT_F_CAM_SENSOR;
+       ret = media_entity_pads_init(&sd->entity, 1, &ov13850->pad);
        if (ret < 0)
                goto err_power_off;
 #endif
```

The driver will probably change in more ways, once I try to merge it into the mainline kernel. But this is currently a  
*WORK IN PROGRESS*...

## 2. Porting the device tree <a name="device_tree"></a>

This step was pretty difficult, first I had to figure out which changes are actually needed. The kernel device tree sources felt a bit confusing as it was unclear to me, which parts are actually required by the device and what those parts depend on. Especially, on the [friendlyElec](https://github.com/friendlyarm/kernel-rockchip/tree/nanopi4-linux-v4.4.y/arch/arm64/boot/dts/rockchip) kernel, I found an abundance of files involved with the rockchip NanoPC-T4 board. That is why I first required some kind of summary, which I found by extracting the compiled device tree blob (.dtb) from the [friendlyElec image](https://drive.google.com/drive/folders/1nFyaZ8mnfjuoXtK6gzpxVhyrdyqKtJfC) and as comparison, taking the `.dtb` from the armbian kernel at `/boot/dtb/`. By using a visual difference tool (like for example [Meld](https://meldmerge.org/)), I was able to see exactly which parts where added by the friendlyElec team. The list includes device tree nodes for:

- DDR timing node
- dummy conventional & virtual Phase Locked Loops (PLL)
- Image sensors OV13850 & OV4689 at the i2c ports 1 & 2
- 2 nodes for the rkisp1 (ISP)
- 2 nodes for the cif_isp (camera interface)
- reboot mode node
- pmu pvtm node
- DFI node
- DMC node
- VPU service node
- IEP node
- 2 ports with a total of 4 endpoints for the mipi-dphy-rx0 node
- mipi-dphy-tx1rx1 with 2 ports and a total of 3 endpoints
- pull down pins for the pwm3a & pwm3b pinctrl nodes
- pinctrl nodes for the ISP & the cam-pins
- board node for the NanoPC-T4
- xin32k fixed clock node
- ... and several others

That is a lot of information, so let's break this down one bite at a time. Which parts do we need for the camera to function?
Let's think about how a camera pipeline works, we have the camera module, which is connected to a MIPI-CSI-2 port via an FCC cable. The MIPI socket is further connected to the ISP, as the ISP needs to get input data from the camera and after further processing and resizing the data is presented to user-space via a `/dev` file (memory interface).
So we can conclude that we only care for the following changes:
- Image sensors OV13850 & OV4689 at the i2c ports 1 & 2
- 2 nodes for the rkisp1 (ISP)
- 2 nodes for the cif_isp (camera interface)
- 2 ports with a total of 4 endpoints for the mipi-dphy-rx0 node (rx0 uses general register files) [\[2\]](https://github.com/friendlyarm/kernel-rockchip/blob/nanopi4-linux-v4.4.y/Documentation/devicetree/bindings/media/rockchip-mipi-dphy.txt#L15)
- mipi-dphy-tx1rx1 with 2 ports and a total of 3 endpoints (tx1rx1 uses its own registers and requires a `reg` entry) [\[2\]](https://github.com/friendlyarm/kernel-rockchip/blob/nanopi4-linux-v4.4.y/Documentation/devicetree/bindings/media/rockchip-mipi-dphy.txt#L15)
- pinctrl nodes for the ISP & the cam-pins

### 2.1 The image sensor

Starting with the target device **OV13850** at the `i2c1` node within the [rk3399-nanopi4-rkisp1.dtsi](https://github.com/friendlyarm/kernel-rockchip/blob/nanopi4-linux-v4.4.y/arch/arm64/boot/dts/rockchip/rk3399-nanopi4-rkisp1.dtsi#L52) file:
```c
ov13850p0: ov13850@10 {
    compatible = "ovti,ov13850";
    status = "okay";
    reg = <0x10>;
    clocks = <&cru SCLK_CIF_OUT>;
    clock-names = "xvclk";

    reset-gpios = <&gpio2 27 GPIO_ACTIVE_HIGH>;
    pwdn-gpios = <&gpio2 28 GPIO_ACTIVE_HIGH>;
    pinctrl-names = "rockchip,camera_default", "rockchip,camera_sleep";
    pinctrl-0 = <&cam0_default_pins &cif_clkout_a>;
    pinctrl-1 = <&cam0_default_pins>;

    port {
        ucam_out0b: endpoint {
            remote-endpoint = <&mipi_in_ucam0b>;
            data-lanes = <1 2>;
        };
    };
};
```
We can identify that the sensor requires a clock, it utilizes GPIO pins 27 and 28 from bank 2 as reset and power-down signal pins. Also, it sets up 2 pinctrl configurations 0 & 1, which contain specific pin configuration for the default and sleep mode. And finally, it creates a port `ucam_out0b` with a remote endpoint `&mipi_in_ucam0b` and two data lanes. But there is also a little mystery with the register `0x10`, let's see if we can figure out why it has to be hexadecimal `0x10` or decimal 16.

We are going to look at the following points:  

1. [The clock](#image-sensor-clock)
2. [The reset and power-down GPIOs](#image-sensor-reset-powerdown)
3. [The pin controls](#image-sensor-pin-controls)
4. [The port and endpoint](#image-sensor-port-endpoint)
5. [The register](#image-sensor-register)
6. [New changes](#dts-new)
7. [Adjusting the device tree](#image-sensor-device-tree)
8. [Testing the changes](#image-sensor-testing)

#### 2.1.1 The clock <a name="image-sensor-clock">

```
clocks = <&cru SCLK_CIF_OUT>;
clock-names = "xvclk";
```
This part refers to the clock used by the driver, why do we need a clock for an image sensor? A brief answer to that question could simply be, to synchronize state changes in memory as most of our hardware works synchonously. (For a longer answer look here: [\[3\]](https://electronics.stackexchange.com/a/93885/267459))  
There are three parts of interest: `&cru`, `SCLK_CIF_OUT` and `xvclk`, the first `&cru` refers to the clock controller of our rk3399 CPU:  

```c
cru: clock-controller@ff760000 {
    compatible = "rockchip,rk3399-cru";
    reg = <0x0 0xff760000 0x0 0x1000>;
    #clock-cells = <1>;
    #reset-cells = <1>;
    assigned-clocks =
        <&cru ACLK_VOP0>, <&cru HCLK_VOP0>,
        ...
    assigned-clock-rates =
         <400000000>,  <200000000>,
         ...
};
```

For more details about the clock controller for the rk3399, look [here](https://git.linuxtv.org/media_tree.git/tree/Documentation/devicetree/bindings/clock/rockchip,rk3399-cru.txt)

The second part `SCLK_CIF_OUT` is about a specific clock used for the image sensor. It has a register ID, set within the [<dt-bindings/clock/rk3399-cru.h>](https://github.com/friendlyarm/kernel-rockchip/blob/nanopi4-linux-v4.4.y/include/dt-bindings/clock/rk3399-cru.h) file.
```c
...
#define SCLK_CIF_OUT			137
...
```
And it is defined as a clock for the camera interface within [cif_isp0 & 1](https://github.com/friendlyarm/kernel-rockchip/blob/nanopi4-linux-v4.4.y/arch/arm64/boot/dts/rockchip/rk3399-linux.dtsi#L89), where it is also assigned to the name `clk_cif_out`.  

Finally, the `xvclk` is the name of the clock to be used by the driver. If we look at the driver for [OV13850](https://github.com/friendlyarm/kernel-rockchip/blob/nanopi4-linux-v4.4.y/drivers/media/i2c/ov13850.c#L1440), we can spot the following line:
```c
...
ov13850->xvclk = devm_clk_get(dev, "xvclk");
...
```

#### 2.1.2 The reset and power-down GPIOs <a name="image-sensor-reset-powerdown">

```c
reset-gpios = <&gpio2 27 GPIO_ACTIVE_HIGH>;
pwdn-gpios = <&gpio2 28 GPIO_ACTIVE_HIGH>;
```

Now we take a closer look at the GPIOs used for reseting and powering down the device. We can locate the reference within the driver:  
```c
...
ov13850->reset_gpio = devm_gpiod_get(dev, "reset", GPIOD_OUT_LOW);
...
ov13850->pwdn_gpio = devm_gpiod_get(dev, "pwdn", GPIOD_OUT_LOW);
...
```

I noticed that the armbian tree doesn't utilize direct pins and instead used macros, for example:  
```c
ep-gpios = <&gpio2 RK_PA4 GPIO_ACTIVE_HIGH>;
```
These macros are defined within the [include/dt-bindings/pinctrl/rockchip.h](https://git.linuxtv.org/media_tree.git/tree/include/dt-bindings/pinctrl/rockchip.h) file.

In a nutshell, I have to add and configure the two GPIOs to the camera sensor, so that the driver can interact with the device by reseting it or powering it down.

#### 2.1.3 The pin controls <a name="image-sensor-port-pin-controls">

```c
pinctrl-names = "rockchip,camera_default", "rockchip,camera_sleep";
pinctrl-0 = <&cam0_default_pins &cif_clkout_a>;
pinctrl-1 = <&cam0_default_pins>;
```

The pin controller is responsible for controlling a set of pins used by hardware. Each pin can be configured for multiple different signals, this is called pin muxing. In our example, we have 2 configurations: `pinctrl-0` and `pinctrl-1`. The names within `pinctrl-names` are assigned to the configuration in order, meaning:
- `rockchip,camera_default` -> `pinctrl-0`
- `rockchip,camera_sleep` -> `pinctrl-1`

More details here [\[4\]](https://git.linuxtv.org/media_tree.git/tree/Documentation/devicetree/bindings/pinctrl/pinctrl-bindings.txt)

The pin configuration can be found within the device tree as a child node of the pinctrl node.  
[arch/arm64/boot/dts/rockchip/rk3399-nanopi4-common.dtsi](https://github.com/friendlyarm/kernel-rockchip/blob/nanopi4-linux-v4.4.y/arch/arm64/boot/dts/rockchip/rk3399-nanopi4-common.dtsi#L1202)
```c
cam_pins {
    cif_clkout_a: cif-clkout-a {
        rockchip,pins = <2 11 RK_FUNC_3 &pcfg_pull_none>;
    };
    ...
    cam0_default_pins: cam0-default-pins {
        rockchip,pins =
            <2 28 RK_FUNC_GPIO &pcfg_pull_down>,
            <2 27 RK_FUNC_GPIO &pcfg_pull_none>;
    };
    ...
};
```

These configurations are then used by the driver like this:

```c
...
#define OF_CAMERA_PINCTRL_STATE_DEFAULT	"rockchip,camera_default"
#define OF_CAMERA_PINCTRL_STATE_SLEEP	"rockchip,camera_sleep"
...
static int ov13850_probe(struct i2c_client *client,
			 const struct i2c_device_id *id)
{
        ...
		ov13850->pins_default =
			pinctrl_lookup_state(ov13850->pinctrl,
					     OF_CAMERA_PINCTRL_STATE_DEFAULT);
        ...
		ov13850->pins_sleep =
			pinctrl_lookup_state(ov13850->pinctrl,
					     OF_CAMERA_PINCTRL_STATE_SLEEP);
        ...
}
...
```

#### 2.1.4 The port & endpoint <a name="image-sensor-port-endpoint"></a>

This part has drastically changed between the two kernel versions, as the connection is now not implemented in form of: sensor -> mipi_dphy -> ISP, but through a simple connection of the sensor and the ISP. The MIPI physical layer protocol is just propagated for the ISP node. More on this part in the [RKISP1 bindings](https://git.linuxtv.org/media_tree.git/tree/Documentation/devicetree/bindings/media/rockchip-isp1.yaml#n163) within the documentation.

Here we define the endpoint within the sensor:

```c
port {
    ucam_out0b: endpoint {
        remote-endpoint = <&mipi_in_ucam0b>;
        data-lanes = <1 2>;
    };
};
```

Here comes the endpoint at the ISP:

```c
&isp0 {
	status = "okay";

	ports {
		port@0 {
			mipi_in_ucam0b: endpoint@0 {
				reg = <0>;
				remote-endpoint = <&ucam_out0b>;
				data-lanes = <1 2>;
			};
		};
	};
```
And at the definition of the `isp0` node, we define the physical layer:
```c
	isp0: isp0@ff910000 {
        ...
		phys = <&mipi_dphy_rx0>;
		phy-names = "dphy";
        ...
```

The port is the data interface of the video device (the point where it connects to other devices) and the configuration for a specific device is called an endpoint.  
As you can see below the output endpoint `ucam_out0b` of the ov13850 camera sensor within the `i2c1` node, is connected to the `isp0` node through the `mipi_in_ucam0b` input endpoint. In the definition of ISP, we assign another node as the MIPI-DPHY physical layer, to transport the data from the sensor to the ISP. The data-lanes property describes that there are two data lanes 1 & 2, which are used for transportation of data, while lane 0 is used for the clock.

The `mipi_dphy_rx0` node is already set up in the mainline kernel, all we have to do is enabling it.

##### What is a MIPI DPHY?

It is a physical layer protocol developed by the [MIPI](https://mipi.org/) alliance. There are several other protocols like the *C-PHY* or the *M-PHY*, used for different purposes. The *D-PHY* protocol is very popular in the automotive and smartphone industry and therefore found within a lot of development boards. A physical layer is the connection point between the camera and the application (camera serial interface **CSI**) or between the display and the application (display serial interface **DSI**).
[\[5\]](https://mipi.org/specifications/d-phy), [\[6\]](https://www.edn.com/all-you-need-to-know-about-mipi-dphy-rx/)  

 The documentation for video interfaces in device trees contains an abundance of information about the connection of multiple video capturing hardware elements with each other. [\[7\]](https://git.linuxtv.org/media_tree.git/tree/Documentation/devicetree/bindings/media/video-interfaces.txt)  
 Especially the example at the bottom is capable of explaining a lot of ideas at once, I highly recommend reading this documentation.

#### 2.1.5 The register <a name="image-sensor-register"></a>

The documentation [\[8\]](https://github.com/friendlyarm/kernel-rockchip/blob/nanopi4-linux-v4.4.y/Documentation/devicetree/bindings/media/rockchip-mipi-dphy.txt#L20) describes the reg entry to be: "offset and length of the register set for the device."

In order to find out more about the correct hexadecimal value to choose let's look at the [data sheet of the OV13850](http://www.t-firefly.com/download/firefly-rk3288/peripherals/Sensor_OV13850-G04A_OmniVision_Specification(V1.1).pdf#page=83&zoom=auto,-274,736). With some help of my mentor, I was able to figure out, that the answer to that question lies within the '*6. register tables*' part. Here the following is stated:  
> The following tables provide descriptions of the device control registers contained in the OV13850. The device slave addresses are `0x20` for write and `0x21` for read when SID= 0, `0x6C` for write and `0x61` for read when SID= 1.

So with SID(select for the serial camera control bus ID) set to 0, the slave addresses are `0x20` (`0010 0000`) and `0x21` (`0010 0001`). The last bit of those two addresses is only used for deciding if it is a read or write address, which means it is not part of the address. Therefore, we can shift the address by 1 to the right and get:
`0010 0000 >> 1 = 0001 0000` and `0010 0001 >> 1 = 0001 0000`, which is `0x10` the offset of our register.

#### 2.1.6 Additional elements that were missing in the old version / removed elements <a name="dts-new"></a>

When we take a look at the [bindings documentation] of the RkISP1, we can see that resets and reset-names are two properties, which are not used anymore. So, let us just drop them.

```c
	resets = <&cru SRST_H_ISP0>, <&cru SRST_ISP0>;
	reset-names = "h_isp0", "isp0";
```

At first, I implemented the image sensor without its power regulators. This works because the system will fallback to dummy regualtors, which bascially means it will act like the regulators are there anyway. But we get some ugly warnings in the kernel log. I was able to implement the regulators by looking at [this example](https://git.linuxtv.org/media_tree.git/tree/arch/arm64/boot/dts/renesas/aistarvision-mipi-adapter-2.1.dtsi), which was kindly pointed out to me by Ezequiel Garcia from Collabora. I took the correct values from the [OV13580 datasheet](http://www.t-firefly.com/download/firefly-rk3288/peripherals/Sensor_OV13850-G04A_OmniVision_Specification(V1.1).pdf#page=26).

```c
ov13850_avdd_2p8v: 2p8v {
    compatible = "regulator-fixed";
    regulator-name = "ov13850_avdd";
    regulator-min-microvolt = <2800000>;
    regulator-max-microvolt = <2800000>;
    regulator-always-on;
};

ov13850_dovdd_1p8v: 1p8v {
    compatible = "regulator-fixed";
    regulator-name = "ov13850_dovdd";
    regulator-min-microvolt = <1800000>;
    regulator-max-microvolt = <1800000>;
    regulator-always-on;
};

ov13850_dvdd_1p2v: 1p2v {
    compatible = "regulator-fixed";
    regulator-name = "ov13850_dvdd";
    regulator-min-microvolt = <1200000>;
    regulator-max-microvolt = <1200000>;
    regulator-always-on;
};
```

#### 2.1.7 Adjusting the device tree <a name="image-sensor-device-tree"></a>

Let us summarize all the previous parts into a set of changes, required for a working camera sensor. First we need to add the image sensor to an I2C bus node, we need to assign a 24MHz clock, the required pins, the register value, as well as the power regulators. Further, we will have to connect the image sensor to the ISP through the MIPI physical layer. In order to have a working MIPI DPHY, it will have to be enabled. Finally, we set up the Image Signal Processor and configure it. From the device tree point of view we should be set and ready to go.

[Here is the complete patch](https://github.com/initBasti/NanoPC-T4_armbian_configuration/blob/main/userpatches/kernel/rockchip64-dev/v5-0001-arm64-dts-Add-OV13850-RkISP1-to-the-device-tree.patch)

#### 2.1.8 Testing the changes <a name="image-sensor-testing"></a>

- Locate the camera on the i2c bus with i2cdetect
```bash
sudo apt-get install i2c-tools
sudo i2cdetect -y 1
```
Without the driver and the device tree properly working it looks like this for me:
```bash
basti@nanopct4:~$ sudo i2cdetect -y 1
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:                         -- -- -- -- -- -- -- -- 
10: -- -- -- -- -- -- -- -- -- -- UU -- -- -- -- -- 
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
40: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
50: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
70: -- -- -- -- -- -- -- --     
```

When everything is fine and dandy, we can see that there is a chip at `i2c-1` address `0x0c`:
```bash
basti@nanopct4:~$ sudo i2cdetect -y 1
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:                         -- -- -- -- 0c -- -- -- 
10: UU -- -- -- -- -- -- -- -- -- UU -- -- -- -- -- 
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
40: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
50: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
70: -- -- -- -- -- -- -- --  
```
- Find the device within the [sysfs](https://en.wikipedia.org/wiki/Sysfs) pseudo file system

When the camera was successfully probed by the driver, it will create an entry within the sysfs. We can track down video devices quite efficiently by looking into the `/sys/class/video4linux` folder.
```bash
basti@nanopct4:~$ ls -l /sys/class/video4linux/
lrwxrwxrwx 1 root root 0 22. Dez 07:26 v4l-subdev0 -> ../../devices/platform/ff910000.isp0/video4linux/v4l-subdev0
lrwxrwxrwx 1 root root 0 22. Dez 07:26 v4l-subdev1 -> ../../devices/platform/ff910000.isp0/video4linux/v4l-subdev1
lrwxrwxrwx 1 root root 0 22. Dez 07:26 v4l-subdev2 -> ../../devices/platform/ff910000.isp0/video4linux/v4l-subdev2
lrwxrwxrwx 1 root root 0 22. Dez 07:26 v4l-subdev3 -> ../../devices/platform/ff110000.i2c/i2c-1/1-0010/video4linux/v4l-subdev3
                                                                                      ^^^^^^^^^^^^^^^^
lrwxrwxrwx 1 root root 0 22. Dez 07:26 video0 -> ../../devices/platform/ff680000.rga/video4linux/video0
lrwxrwxrwx 1 root root 0 22. Dez 07:26 video1 -> ../../devices/platform/ff910000.isp0/video4linux/video1
lrwxrwxrwx 1 root root 0 22. Dez 07:26 video2 -> ../../devices/platform/ff910000.isp0/video4linux/video2
lrwxrwxrwx 1 root root 0 22. Dez 07:26 video3 -> ../../devices/platform/ff910000.isp0/video4linux/video3
lrwxrwxrwx 1 root root 0 22. Dez 07:26 video4 -> ../../devices/platform/ff910000.isp0/video4linux/video4
lrwxrwxrwx 1 root root 0 22. Dez 07:26 video5 -> ../../devices/platform/ff660000.video-codec/video4linux/video5
lrwxrwxrwx 1 root root 0 22. Dez 07:26 video6 -> ../../devices/platform/ff650000.video-codec/video4linux/video6
lrwxrwxrwx 1 root root 0 22. Dez 07:26 video7 -> ../../devices/platform/ff650000.video-codec/video4linux/video7
basti@nanopct4:~$ cat /sys/class/video4linux/v4l-subdev3/name 
ov13850 1-0010
```
- check dmesg for output

The driver will also notify it self to the kernel log, when it successfully detected the camera sensor:
```bash
sudo dmesg
...
[    7.253238] ov13850 1-0010: driver version: 00.01.01
...
[    7.258922] ov13850 1-0010: Detected OV00d850 sensor, REVISION 0xb1
...
```

## 3. Use the latest kernel and test the camera <a name="armbian_image"></a>

Within [this project](https://github.com/initBasti/NanoPC-T4_armbian_configuration), I have explained in great detail how to create an image for the NanoPC-T4 with the latest and greatest linux kernel sources. I have utilized the tool armbian for this job. At the bottom of the description, I describe how to test the camera as well.

The tests boil down to the following checklist:
* Create a camera pipeline by negotiating the formats across all devices in the chain.
* Capture some amount of frames with either `V4L2-ctl`/`cam`/`gstreamer` etc.
* Convert the raw video material to `.mp4` with `ffmpeg` or watch directly with `ffplay`

[https://github.com/initBasti/NanoPC-T4_armbian_configuration](https://github.com/initBasti/NanoPC-T4_armbian_configuration)

## 4. Sending it to the mainline kernel <a name="patching"></a>

Work in progress..

## References
[\[1\] Wiki article about the camera from the manufacturer](http://wiki.friendlyarm.com/wiki/index.php/Matrix_-_CAM1320)  
[\[2\] Documentation about mipi-dphy devicetree nodes](https://github.com/friendlyarm/kernel-rockchip/blob/nanopi4-linux-v4.4.y/Documentation/devicetree/bindings/media/rockchip-mipi-dphy.txt#L15)  
[\[3\] Electronics stachexchange answer about the need for clocks in computing by the user Alfred Centauri](https://electronics.stackexchange.com/a/93885/267459)  
[\[4\] Documentation about pinctrl bindings](https://git.linuxtv.org/media_tree.git/tree/Documentation/devicetree/bindings/pinctrl/pinctrl-bindings.txt)  
[\[5\] MIPI D-PHY specification on mipi.org](https://mipi.org/specifications/d-phy)  
[\[6\] Great article about MIPI D-PHY by Love Gupta on EDN.com](https://www.edn.com/all-you-need-to-know-about-mipi-dphy-rx/)  
[\[7\] Documentation about how to write video interfaces](https://git.linuxtv.org/media_tree.git/tree/Documentation/devicetree/bindings/media/video-interfaces.txt)  
[\[8\] Documentation about the `reg` entry](https://github.com/friendlyarm/kernel-rockchip/blob/nanopi4-linux-v4.4.y/Documentation/devicetree/bindings/media/rockchip-mipi-dphy.txt#L20)  
