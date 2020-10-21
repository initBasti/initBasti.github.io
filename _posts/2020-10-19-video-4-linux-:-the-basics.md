# Getting into Video 4 Linux development

## What is Video 4 Linux?

A framework used for video streaming, control, capturing, and codec support, radio tuning and modulating and even Radio Data Systems (RDS) (Radio traffic information).

### Which problem does it solve?

Before V4L (Video 4 Linux, the current version is called V4L2), there were a lot of very complex drivers, that were difficult to work with. So the goal of this framework is to create a common pathway for the creation of drivers as well as a universal API, which userspace can use in order to work with different drivers.

---

## Table of contents

1. [Exploring the basics](#basics)
2. [How the digital data from the hardware device is handled by the software driver?](#streaming)

---

## Exploring the basics <a name="basics"></a>

* What is an image and how does it differ from a video?
* How is the image data received?
* How is color represented?
* And what processing is performed after that until we can see it on the monitor?

There is a great course on GitHub, about this topic, if you would like to dive in a bit deeper. [\[1\]](https://github.com/leandromoreira/digital_video_introduction#intro)

The general process works like this:

The incoming light waves from the subject are focused on an image sensor through a lens. The image sensor consists of several photodiodes that generate an analog signal which is converted into a digital signal. To capture colors, a color filter array is placed over the image sensor in a specific pattern, so that each sensor element only captures green, blue, or red light waves. The resulting information is used in an interpolation algorithm that calculates the missing color values of all pixels for which color values were discarded. The result is polished and encoded through further processing in order to make it easy to transfer and save.

### Basic concept of an image

The image, which you can see on your computer monitor, TV screen or projector, is made of single pixels. Each pixel contains information about its ratio of **R**ed, **G**reen, and **B**lue (atleast with the RGB color model), which make up the visible color. There are raster and vector images, usually the term digital image refers to raster images. A raster image is an array with a fixed amount of rows and columns, where each element in the grid contains information about the brightness of the color.

#### What is a pixel?

The term pixel is used quite frequently in multiple contexts, but generally refers to:
* The smallest controllable element of a picture on a screen [\[2\]](https://en.wikipedia.org/wiki/Pixel)
* A single point within a raster image matrix is called a pixel (picture element). One pixel represents the intensity (usually a numeric value) of a given color. [\[3\]](https://github.com/leandromoreira/digital_video_introduction#basic-terminology)
* A synonym for the term sensel, which refers to a single photodiode within a image sensor

#### Difference between still image and video

At a very simple level of abstraction, the difference between a still image and a video is simply that there are multiple frames per second that are required to create the illusion of movement for our eyes. But as we'll find out later, it is going to take a little more effort to make watching and streaming videos a nice experience.

Next I'll talk about how hardware actually receives the information required for building an image.

### What is an image sensor?

A device that is used to detect information usable for creating images. It uses photodiodes to capture light and transform it to an electric signal.

#### What is a photodiode?

A photodiode works quite similar to a normal diode, which is an electrical component that ideally has an infinitely small resistance in one direction and an infinitely big resistance in the other direction (therefore directing a flow of current). It uses a [P-N junction](https://en.wikipedia.org/wiki/P%E2%80%93n_junction) of two semiconductors (mostly silicon), meaning one side is over-saturated with electrons and the other is lacking electrons. The photodiode has been depleted of mobile charge carriers (by applying [reverse bias](https://en.wikipedia.org/wiki/P%E2%80%93n_junction#Reverse_bias)), which creates an electric field, which when hit by a photon (e-), hole(h+) pair attracts the photon to the positive charged N-layer and the positive hole charge to the P-layer. The energy of the photon has to be big enough, to attract an electron across the depletion region of the semiconductor, which in the case of silicon fits every light-wave with a wave-length shorter or equal to near-infrared light. That current flow is then amplified and later converted into a digital signal.

This is just a quick & dirty summary, if you want to go deeper into this topic start with: [\[4\]](https://youtu.be/4Deyx3RighA).

![Photodiode schematic](https://www.rp-photonics.com/img/pin_photodiode.png) [\[www.rp-photonics.com\]](https://www.rp-photonics.com/photodiodes.html)

#### What are common types of image sensors?

CCD (Charge Coupled Device)
* Each cell of the CCD is an analog device, which generates an electric charge that is shifted towards the single amplifier for the column.
* the signal has to be converted to a digital signal with an ADC (Analog to Digital Converter).
* Used for high-quality video cameras as they produce less noise than a CMOS

CMOS (Complementary Metal Oxid Semiconductor)
* Uses an amplifier for each sensor cell together with micro lenses in order to not lose information of photons hitting the amplifier instead of the cell.
* Cheaper and more commonly used, with a lower power consumption than a CCD.

For more information refer to: [\[5\]](https://en.wikipedia.org/wiki/Image_sensor)


Now we know how, the image data is perceived by the device, how does it obtain the color information though?

### Obtaining the color information for each pixel

We have a little problem, if we would just measure the energy level of a particular light beam with each photosensor, we end up with a black and white image. That is because the single sensor element cannot process the wavelength of the light only it's intesity. (look here if you want to know more about what color actually is [\[6\]](https://en.wikipedia.org/wiki/Color) or here [\[7\]](http://poynton.ca/PDFs/ColorFAQ.pdf), if more is not enough)

#### Color model

There are different representations of color used by various devices, the most commonly used model is the additive color model **RGB**, which represents color as a combination of the respective brightness levels of *red*, *green*, and *blue*. You can find RGB in most electronic input devices like video cameras, digital cameras or image scanners, and also in most electronic output devices like LCD, plasma, OLED, and other displays.  
Another type of color model is **CMY** & **CMYK**, those are primarily used for color printing devices, as they are subtractive color models. which means that a combination of all three colors *cyan*, *magenta* and *yellow* results in *black*, while in RGB the combination of *red*, *green* and *blue* results in *white*. This is particularly useful because the natural background color of paper is *white* and therefore you cannot print *black* color by using a combination of 0 *red*, 0 *green*, and 0 *blue*.  
A commonly used color model is **YCbCr**, which works by using coordinate transformation on the original RGB data. The result is a mix of the [luma](https://en.wikipedia.org/wiki/Luma_(video))(Y) together with a combination of the blue-difference to green (Cb) & red difference to green (Cr), as our eyes are more sensitive to brightness than to color, we can use this model for image processing in order to reduce unnecessary information more effectively.

#### Color filter array

One way to address this problem is to create a grid of pure red, green, and blue pixels by filtering out specific wavelengths from a specific group of photodiodes.  
Example (Bayer filter array):

![Bayer filter](https://external-content.duckduckgo.com/iu/?u=https%3A%2F%2Fupload.wikimedia.org%2Fwikipedia%2Fcommons%2Fthumb%2Fe%2Fef%2FBayer_matrix.svg%2F220px-Bayer_matrix.svg.png&f=1&nofb=1)

Each index represents a photodiode with a filter, which can only be overcome by a certain wavelength (color). We can use this information along with an interpolation algorithm to calculate the color values of the missing (discarded) values.

##### Bayer filter

The bayer filter is a color filter pattern with 50% Green, 25% red & 25% blue filters above the photosensors. (The reason for the portion of green filters is the way our eyes work, as they are more sensitive to green light during daytime) [\[8\]](https://en.wikipedia.org/wiki/Bayer_filter)  
It is one of the most used variations of the color filter array and multiple new variations are based on it's design. (Example: [X-trans](https://en.wikipedia.org/wiki/Fujifilm_X-Trans_sensor)).

#### Demosaicing/Interpolation

As explained above, we need to calculate the missing values for each sensor element. For example a photodiode with a blue filter, discarded all the red and green light information and only measured the brightness of the blue light. There are multiple different algorithms used in order to calculate the red and green light brightness of that pixel. The choice off the algorithm depends heavily on the use case, if you just want to make a quick preview of an image and you don't care too much about the quality a simple bi-linear interpolation algorithm will do just fine. But the more quality you want and the more robustness (ability to adjust to multiple different scenarios) you require, the more complex the algorithm will be.

Here is a small example of how these values can be calculated: [\[9\]](https://www.cse.iitb.ac.in/~ajitvr/CS663_Fall2016/demosaicing.pdf)

### Making video data usable through data compression

If a video runs at 30 frames per second and has a resolution of 1080p (1920 * 1080 pixels = 2073600 pixels), each 24-bit pixel requires 3 bytes to store color information. Then we're talking about 6.2MB per frame * 30 = 186MB per second. If you were to watch a 3 minute video, the entire video would be 33.5 GB in size. And this isn't even ultra high quality video (134.3GB). So the question is, how can we effectively use video with limited options for storage and data transfer, especially over the Internet.

#### video compression

This is a method of reducing the size of a set of frames to make it easier to transfer / store the data. There are several algorithms that strive for different goals for different use cases.
For example, if the majority of your image is exactly the same color, a **spatial compression** algorithm can reduce the size of the data stored by identifying blocks in which no color change is detected. Or there can be a number of frames within a video where the majority of the pixels do not change between frames. In this case, a **temporal compression** algorithm can reduce the size of the video by only storing the difference between several frames and the respective key-frame(intra frame).  

Then there is **chroma subsampling** which reduces the total amount of pixels storing color information by reusing the color information of neighboring pixels. Important about this transformation is that we keep all the brightness information because our eyes are much more sensitive to brightness than to color (the Y'CbCr color model is very useful for that case). Common pixel formats for chroma sub-sampling are 4:4:4, 4:4:2 & 4:2:0 [\[10\]](https://en.wikipedia.org/wiki/Chroma_subsampling#How_subsampling_works), The first number describes how many pixels per row are used for sampling (horizontal length), the second number describes how much color information is stored in the first line, and the third number indicates how many color changes are recorded from the first to the second line. For example in 4: 4: 2, we get the colors from a width of 4 pixels and keep all color information from the first line, but only store 2 color information from the second line and use the above color value for the discarded values. We therefore reduce the color information by 2/8, while 4: 2: 0 would reduce the color information to 50%.  

A simple way to reduce the size of an image is to reduce only the [**bit depth**](https://en.wikipedia.org/wiki/Color_depth) of the image, which describes how many different levels of brightness from red to green and blue can be stored in a single pixel, for example: 24-bit color information can store 2 ^ 8 (256) tones of each RGB color, resulting in about 16 million different color combinations at 3 bytes per pixel in size. If you reduced this color information to a total of 16 bits (5 bits per color, 1 bit for transparency with 32768 color combinations), you could save 1/3 of the total size.  

The techniques mentioned are lossy algorithms which lose information when the size is reduced. There are also lossless algorithms that mainly perform statistical analysis on images to find and remove redundancy. They are great when we can't afford to lose information from the picture (for example for medical pictures / videos). However, they require significantly more storage space, which is why most of the known video processing operations use lossy algorithms.

#### Temporal compression

##### I/B/P Frames

- I-Frames: Intra-coded pictures
    * An I-frame doesn't rely on any other image in order to be rendered, it is the static part of the video which is referenced by P- & B-frames
- P-Frames: Predicted pictures
    * difference between the target picture and the last I-frame
- B-Frames: Bidirectional predicted picture
    * Utilizes past & future references to the last P-frame and the next I-frame to compose a picture

###### Question: Why does a video conference never need B-frames?

Because it cannot reference images in the future as it is a live streaming of data.


#### What are codecs?

Codecs are a short form for compressor-decompressor, they are used to compress the size of images, while keeping as much information as possible. Or to decompress an image in order to use it. There are multiple codecs for different purposes. For example for video capturing, the goal is to save the greatest possible amount of information, as you are always able to compress the image afterwards but you will not be able to recover lost information. When editing a video you want a codec that enables you to swiftly move inbetween frames without having to sample together a frame from different parts of other frames. Then for playback and transmission of videos, we generally want to have a good balance out of the lowest possible size paired with a good looking quality. And finally for archival, you want to keep as much information as possible, in order to use that data later on.  

Source: [\[11\]](https://youtu.be/sisvOeZItb0)

There are software and hardware codecs, which both have their advantages and disadvantages. A hardware codec is generally quite fast at doing a very specific job, most of them are very expensive and provide a high quality output. Software codecs shine because of their flexibility paired with an easier installation and possiblity for end-to-end encryption.  

##### Popular codec formats

* [HEVC/H.265](https://en.wikipedia.org/wiki/High_Efficiency_Video_Coding) (25-50% better compression than H.264 at the same quality)
* [H.264](https://en.wikipedia.org/wiki/Advanced_Video_Coding) (lossless & lossy variations)
* [RealVideo](https://en.wikipedia.org/wiki/RealVideo) (proprietary)
* [DivX](https://en.wikipedia.org/wiki/DivX) (freemium, windows/mac only)

##### What is a codec device?

* A software or hardware device, that converts a RAW byte-stream into streamable digital data or vice-versa digital data into RAW byte-stream
* The codec recieves a byte-stream, parses it to extract meta data and decodes it

##### Examples for devices

* Software codec devices:
    + OBS Studio [\[12\]](https://obsproject.com/)

* Hardware codec devices:
    + Video decoder used for connecting a camera directly to a monitor through the decoder. (Example: [Axis T8705](https://www.axis.com/files/datasheet/ds_t8705_decoder_t10110350_en_2010.pdf))
    + A video encoder used for streaming video & audio data to a internet streaming provider in the correct format. (Example: [Digicast DMB-8800A](http://www.digicast.cn/en/proddetail.asp?id=416))

##### Different types of hardware codecs

- stateful hardware codecs:

    * the different states of the steps for decoding a byte-stream are kept in the hardware/firmware
    * user-space only has to provide the RAW/compressed byte-stream data to the codec hardware

![Stateful decoder working principle by Maxime Rippard](https://raw.githubusercontent.com/initBasti/initBasti.github.io/master/images/stateful_decoder_diagram.jpg)
[\[13\]](https://events19.linuxfoundation.org/wp-content/uploads/2017/12/Supporting-Hardware-Codecs-in-a-Linux-System-Maxime-Ripard-Bootlin.pdf)

- stateless hardware codecs:

    * does not retain any state within it's hardware or firmware
    * user-space is responsible for maintaining the processed state of each frame and deliver it to the driver
    * user-space also has to make sure that the order is correct
    * hardware only provides the decoding of the byte-stream

![Stateless decoder working principle by Maxime Rippard](https://raw.githubusercontent.com/initBasti/initBasti.github.io/master/images/stateless_decoder_diagram.jpg)
[\[13\]](https://events19.linuxfoundation.org/wp-content/uploads/2017/12/Supporting-Hardware-Codecs-in-a-Linux-System-Maxime-Ripard-Bootlin.pdf)

#### Further image processing

* white balance
* crop / resizing
* exposure
* image stabilization
* image effects
* flip / rotate

### How is the digital data from the hardware device processed by software drivers? <a name="streaming"></a>

#### Observing a video stream process

Let's look at an example of my webcam streaming for a few seconds. We can watch that process through an invocation of `strace v4l2-ctl -d /dev/video0 --stream-mmap --stream-count 10`, at first we will determine, what that command actually does.  
The *strace* command is a Linux utility for tracing the system-calls of a program, which can be very useful for debugging or in our case for investigating the internals. The program *v4l2-ctl* is part of the [v4l-utils](https://git.linuxtv.org/v4l-utils.git) library & applications package, it is used to interact with video 4 linux drivers directly over the command line. The parameter `--stream-mmap` and `--stream-count 10` translate into: Get 10 frames of video from the device, which in this case is /dev/video0.  

More specifically `--stream-mmap` says don't copy the memory over to the application but use the pointer to the buffer within the driver, by mapping the memory from the device into the application's address space. [\[14\]](https://www.kernel.org/doc/html/latest/userspace-api/media/v4l/mmap.html#mmap) The alternatives to `--stream-mmap` are `--stream-user`, which allocates the buffers only within the application [\[15\]](https://www.kernel.org/doc/html/latest/userspace-api/media/v4l/userp.html) and `--stream-dmabuf` to either create a file-descriptor to share data with other devices (exporter-role) or use a DMA buffer file-descriptor from another device (importer-role). [\[16\]](https://www.kernel.org/doc/html/latest/userspace-api/media/v4l/dmabuf.html)  

Looking at the output, we can quickly spot a certain pattern, namely there are a lot of `ioctl` system calls, with parameters that start with `VIDIOC_*`.  

#### What is the `ioctl` system call and what does it do?  

It is a way for user-space to make a direct request to the operating system, for a functionality which is not easily expressed in terms of the standard system calls (read, write, open, fork, etc.). Especially device drivers often require a certain very specific utility for interacting with the device and it would be very inefficient to implement a separate system call into the kernel API for that purpose.  
The basic call looks like this: `int ioctl(int fd, unsigned long request, ...)`, where `fd` is the file descriptor (index within the open files table of the current process), which points to a device file. And `request` is a an integer, that fits to a specific function within that drivers ioctl operations. Finally, the three dots are C syntax for variable arguments, which are mostly used in v4l2 to pass a pointer to buffers.  

One small example:  
In the strace output, I am able to spot the following line:  
`ioctl(3, VIDIOC_REQBUFS, {type=V4L2_BUF_TYPE_VIDEO_CAPTURE, memory=V4L2_MEMORY_MMAP, count=4 => 4}) = 0`
* 3 is the file descriptor
* VIDIOC_REQBUFS is the request number 8 (located at your kernel source under [include/uapi/linux/videodev2.h](https://elixir.bootlin.com/linux/latest/source/include/uapi/linux/videodev2.h#L2473) and in the v4l-utils source under `include/linux/videodev2`)
* And the last part is a struct instance of `struct v4l2_requestbuffers`, where we define the type, memory location (see above `--stream-mmap`, `--stream-user` etc) and the amount (examine this in the v4l-utils source under: `utils/common/v4l-helpers.h`)  

Device drivers mostly implement their own version of this `ioctl`, for example in the stkwebcam driver (kernel source: [drivers/media/usb/stkwebcam](https://elixir.bootlin.com/linux/latest/source/drivers/media/usb/stkwebcam/stk-webcam.c#L1031)), which is then assigned to the specific `ioctl` request number via the `struct v4l2_ioctl_ops` structure of function pointers.  

#### Looking at the details

Now that we know, what the command does and what those `ioctl` system calls are, let's look at the details of the streaming process.  

* Start the application & load all the required libraries
* check if the device exists
* [ioctl **VIDIOC_QUERYCAP**] : *Query the device capabilities*
    - Check if the device is compatible to the V4L API
    - It explains which actions are possible with the device, for example which interfaces or image formats it supports

* [ioctl **VIDIOC_QUERYCTL & VIDIOC_QUERY_EXT_CTRL**] : *Query the device controls*
    - Devices provide certain controls, where the user can perform changes to settings like: brightness, hue, gamma, sharpness etc.
    - Request the current settings from the device or to set the values.

* [ioctl **VIDIOC_G_SELECTION**] : *Get the selections within the frames*
    - Get information about any active selections within the video (used for croping)

* [ioctl **VIDIOC_SUBSCRIBE_EVENT**] : *Tell the system which events you want to listen to*
    - Used in [streaming_set_cap](https://git.linuxtv.org/v4l-utils.git/tree/utils/v4l2-ctl/v4l2-ctl-streaming.cpp#n1757) in order to listen to the V4L_EVENT_EOS (End of stream event), which is not supported by every driver.
    - What kind of events are there? [\[17\]](https://www.kernel.org/doc/html/v4.12/media/uapi/v4l/vidioc-dqevent.html#event-type)
        * Vertical sync is performed
        * End of stream reached
        * A control (brightness, gamma, contrast etc.) was changed either at the device or through the flags
        * Receiving a frame from the camera
        * Parameter change at runtime (resolution change, format change from the input etc.)
        * Motion detected while capturing video
        * Or a driver private event (custom)

* [ioctl **VIDIOC_G_INPUT**] : *Get the index of the input source*
    - fills the provided integer with the index of the input source

* [ioctl **VIDIOC_ENUM_INPUT**] : *Get information from the input source*
    - There are three possible input sources:
        * Tuner
        * Video device (for example a camera sensor or HDMI)
        * Touch devices (touch screen, touch pad etc.)
    - Get information about the status, the name, possible settings and the used video standard

* [ioctl **VIDIOC_REQBUFS**] : *Set up the buffers to stream the data from the device*
    - The buffers are created with the following options: *buffer type*, *type of memory mapping* and *count of buffers*
    - There are multiple different types of buffers from *capture* to *output*, *radio* and special types. The interesting part for us are the *capture* devices. The table of buffer types is documented here [\[18\]](https://www.kernel.org/doc/html/v4.12/media/uapi/v4l/buffer.html#c.v4l2_buf_type).
    - The memory mapping type is chosen by the user from the command line (mmap, userptr or dma)
    - The amount of buffers is usually 4 but is variable, when the given amount is 0 all mapped buffers are freed up

* [ioctl **VIDIOC_QUERYBUF**] : *Check the status of a buffer*
    - The following states of a buffer are of interest to us: *mapped*, *enqueued*, *full* or *empty* [\[19\]](https://www.kernel.org/doc/html/v4.12/media/uapi/v4l/mmap.html#streaming-i-o-memory-mapping)
    - The status of a buffer can be observed at any time after being requested
    - In this example QUERYBUF is used to get the offsets for the buffers

* [ioctl **VIDIOC_QBUF**] : *Add an empty capture buffer to the incoming queue*
    - enqueue empty (capture) or filled (output) buffer in the driver's incoming queue
    - the buffer is then filled with data by the camera device, after which it is placed into the outgoing queue
    - the data is not copied, only pointers are exchanged

* [ioctl **VIDIOC_G_FMT**] : *Get the current pixel format settings of the driver*
    - window format, pixel format(RGB,YUV,etc.), capture field, color space [\[20\]](https://www.kernel.org/doc/html/v4.12/media/uapi/v4l/pixfmt-006.html#c.v4l2_colorspace)
    - an application should always get the format first with *G_FMT* before changing the format with *S_FMT*

* [ioctl **VIDIOC_STREAMON**] : *Start the streaming process*
    - activate capture devices and start filling input buffers
    - calling STREAMON while streaming has no effect

* [ioctl **VIDIOC_DQBUF**] : *Remove a filled capture buffer from the outgoing queue*
    - remove the buffer from the outgoing queue of the driver after the driver handled the I/O of the capture device
    - if no buffer is found in the outgoing queue DQBUF will immediatly return while raising the EAGAIN error code (used for non-blocking I/O to notify that no data was available)

* [ioctl **VIDIOC_STREAMOFF**] : *Stop the streaming process*
    - remove any buffer from the incoming queue and outgoing queue (all non-dequeued buffers are lost)
    - resets everything to state after calling REQBUFS

##### Summary

Let's wrap this up, with a short summary of what just happend:  

After we have dealt with all the built up of the application, the streaming starts by getting the capabilities of the camera device (Check what you can or cannot do with the device).  [*QUERYCAP*]  

Then we perform a quick check for the current settings at the device, and get any preconfigured selection of the capture area.  [*QUERYCTL* & *QUERY_EXT_CTRL*] and [*G_SELECTION*]  

We make sure that we listen to the 'End of stream' event and get the input source from the driver as well as information about it.  [*SUBSCRIBE_EVENT*] and [*G_INPUT* & *ENUM_INPUT*]  

At this point, we are ready to work with the buffers, first we set them up with the demanded type (capture) as well as the memory mapping type (mmap through --stream-mmap). Followed by getting the current status and offsets from buffers at the driver, those buffers are then mapped into user-space with a few calls to the `mmap` syscall (within the [obtain_bufs method](https://git.linuxtv.org/v4l-utils.git/tree/utils/v4l2-ctl/v4l2-ctl-streaming.cpp#n1804)).  [*REQBUFS*] and [*QUERYBUF*]  

Now, we can queue those buffers into the driver's incoming queue, notice that the capture device is deactivated until STREAMON is called. [*QBUF*]  

Next the window & pixel format is pulled from the driver in order to interpret the image data correctly. [*G_FMT*]  

As mentioned before, a call to STREAMON starts the streaming process, the capture device is activated and the driver handles the incoming I/O. [*STREAMON*]  

The application dequeues buffers filled with data from the device of the driver's outgoing queue. After it has processed the data it queues the buffer back into the incoming queue. This process is continued until either the 'End of stream' event is raised by the driver, the application calls STREAMOFF or until the application is killed. [*DQ_BUF* & *STREAMOFF*]  

![I/O states](https://static.lwn.net/images/ns/kernel/v4l2_buffers.png)
[\[21\]](https://lwn.net/Articles/240667/)

---

## Reference
[\[1\] In depth introduction to digital video by Leandro Moreira](https://github.com/leandromoreira/digital_video_introduction#intro)  
[\[2\] Article about pixels on wikipedia](https://en.wikipedia.org/wiki/Pixel)  
[\[3\] Basic terminology for digital images by Leandro Moreira](https://github.com/leandromoreira/digital_video_introduction#basic-terminology)  
[\[4\] very good video series about image sensors by Blake Jacquot](https://youtu.be/4Deyx3RighA)  
[\[5\] in-depth article about image sensors on wikipedia](https://en.wikipedia.org/wiki/Image_sensor)  
[\[6\] Description of how our eyes process color](https://en.wikipedia.org/wiki/Color)  
[\[7\] Frequently asked questions about color by charles poynton](http://poynton.ca/PDFs/ColorFAQ.pdf)  
[\[8\] Explanation of the bayer filter and reasons for it's design](https://en.wikipedia.org/wiki/Bayer_filter)  
[\[9\] Set of slides about demosaicing with formulas for calculation](https://www.cse.iitb.ac.in/~ajitvr/CS663_Fall2016/demosaicing.pdf)  
[\[10\] Visual representation of chroma-subsampling formats on wikipedia](https://en.wikipedia.org/wiki/Chroma_subsampling#How_subsampling_works)  
[\[11\] Quick basic explanation of codecs by B&H Photo Video](https://youtu.be/sisvOeZItb0)  
[\[12\] OBS studio - broadcast software](https://obsproject.com/)  
[\[13\] Presentation by Maxime Ripard about codecs in Linux](https://events19.linuxfoundation.org/wp-content/uploads/2017/12/Supporting-Hardware-Codecs-in-a-Linux-System-Maxime-Ripard-Bootlin.pdf)  
[\[14\] Kernel documentation about memory mapped buffers for streaming](https://www.kernel.org/doc/html/latest/userspace-api/media/v4l/mmap.html#mmap)  
[\[15\] Kernel documentation about user-space mapped buffers for streaming](https://www.kernel.org/doc/html/latest/userspace-api/media/v4l/userp.html)  
[\[16\] Kernel documentation about DMA buffers for streaming](https://www.kernel.org/doc/html/latest/userspace-api/media/v4l/dmabuf.html)  
[\[17\] Kernel documentation: event types](https://www.kernel.org/doc/html/v4.12/media/uapi/v4l/vidioc-dqevent.html#event-type)  
[\[18\] Kernel documentation buffer types](https://www.kernel.org/doc/html/v4.12/media/uapi/v4l/buffer.html#c.v4l2_buf_type)  
[\[19\] Kernel documentation explaination of the streaming process (at the bottom)](https://www.kernel.org/doc/html/v4.12/media/uapi/v4l/mmap.html#streaming-i-o-memory-mapping)  
[\[20\] Kernel documentation available color spaces](https://www.kernel.org/doc/html/v4.12/media/uapi/v4l/pixfmt-006.html#c.v4l2_colorspace)  
[\[21\] Great explaination of this process on LWN.net by Jonathan Corbet](https://lwn.net/Articles/240667/)  
