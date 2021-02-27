## An analysis of the RAW bayer images with the OV13850 image sensor on the RkISP1 platform

### Introduction

I am currently working on porting and improving the driver for the OV13850 image sensor. The image sensor was originally built by the Rockchip team and I try to upstream an improved version of it with a few extra features. If you want to know more about this you can take a look at this [post](https://sebastianfricke.me/porting-the-ov13850-camera/).
One of those new features is the ability to flip the image horizontally and/or vertically. I was asked if the Bayer order changes on those flips during the patch review and was not able to find a clear indication within the [data-sheet](http://download.t-firefly.com/product/RK3288/Docs/Peripherals/OV13850%20datasheet/Sensor_OV13850-G04A_OmniVision_SpecificationV1.pdf). This small post is dedicated to documenting my journey of finding the answer to that question.

##### Why is it important to know if the Bayer order changes during a flip?

Because the order of the Bayer pattern is critical for the choice of the output Pixel-format.

### Data-sheet

There is one data-sheet available online for this sensor, which mentions that the color filters are arranged in a Bayer pattern and that there is a line-alternating arrangement of the primary color **BG/GR** array. Additionally, it has a comment about flips, which states that the **readout order of the sensor is reversed either horizontally or vertically** depending on the flip variant.

When I think about that sentence, my initial thought was like this:

**without flips:**  
BGBG BGBG BGBG BGBG  
GRGR GRGR GRGR GRGR  
BGBG BGBG BGBG BGBG  
GRGR GRGR GRGR GRGR  
*(SBGGR10)*

**When we now reverse the columns (vertical flip) we get:**  
GRGR GRGR GRGR GRGR  
BGBG BGBG BGBG BGBG  
GRGR GRGR GRGR GRGR  
BGBG BGBG BGBG BGBG  
*(SGRBG10)*

**with a horizontal flip:**  
GBGB GBGB GBGB GBGB  
RGRG RGRG RGRG RGRG  
GBGB GBGB GBGB GBGB  
RGRG RGRG RGRG RGRG  
*(SGBRG10)*

**with horizontal flip and vertical flip:**  
RGRG RGRG RGRG RGRG  
GBGB GBGB GBGB GBGB  
RGRG RGRG RGRG RGRG  
GBGB GBGB GBGB GBGB  
*(SRGGB10)*

So I assume that the bayer order should change, next I try to prove it by viewing the raw images with different flips and check for anomalies.


### Choosing the right tool

In order to find proof that the Bayer order does change, I will try to capture RAW video from the camera pipeline and observe the result. I will capture every frame with the sensor format `SRGGB10` and look at the result. My thought process behind this is that if the order changes then we should observe some weird colors as the brightness values of one color are used for a different color.

I found a few tools, that seemed to be viable for that job and a few that are currently not ready. I am currently working a lot with the `cam` application from [libcamera](https://www.libcamera.org/), but I noticed that it is currently not ready to capture raw video from the pipeline as the `Raw` stream role isn't implemented yet. So I used `v4l2-ctl` and `yavta` in order to capture raw frames. For both, I have to configure the camera pipeline in advance and I explain how I did it in the next section. After the recording of the video, I needed to find a way to view those videos and extract a frame. I found the [vooya](https://www.offminor.de/) raw video player to be very helpful for that job as well as the [rawpixels.net](http://rawpixels.net/) tool as it provided a few more format options than *vooya*.

### Configuring the tool to stream RAW videos

The camera pipeline for the RkISP1 platform looks like this:
![RkISP1 platform](rkisp1_pipeline.jpg)

Most of the green nodes have one or more sinks (input) and at least one or more sources (output). At first, we activate the connection between the image signal processor and the image sensor.
```bash
# Reset all links and configuration first
media-ctl --device /dev/media0 --reset
media-ctl --device /dev/media0 --links "'ov13850 1-0010':0 -> 'rkisp1_isp':0 [1]"
```

The RkISP1 has two paths for the processed image data, the *main* and the *self* path, these can be used for instance to present a low-resolution preview video through the self-path and a high-resolution video through the main-path. We are going to configure the pipeline to use the *main-path* with the following command:
```bash
media-ctl --device /dev/media0 --links "'rkisp1_isp':2 -> 'rkisp1_resizer_mainpath':0 [1]"
```

Now we have the sub-devices connected, within the next step, we will set the image format for the sub-devices.

```bash
media-ctl --device /dev/media0 --set-v4l2 '"ov13850 1-0010":0 [fmt:SBGGR10_1X10/2112x1568]'
media-ctl --device /dev/media0 --set-v4l2 '"rkisp1_isp":0 [fmt:SBGGR10_1X10/2112x1568 crop: (0,0)/2112x1568]'
media-ctl --device /dev/media0 --set-v4l2 '"rkisp1_isp":2 [fmt:SBGGR10_1X10/2112x1568 crop: (0,0)/2112x1568]'
media-ctl --device /dev/media0 --set-v4l2 '"rkisp1_resizer_mainpath":0 [fmt:SBGGR10_1X10/2112x1568]'
media-ctl --device /dev/media0 --set-v4l2 '"rkisp1_resizer_mainpath":1 [fmt:SBGGR10_1X10/2112x1568]'
```

And finally, we configure the DMA-engine, which is the point where we get the image data.

```bash
v4l2-ctl --device /dev/video0 --set-fmt-video "width=2112,height=1568,pixelformat=BG10"
```

We can figure out the correct pixelformat with the following command:

```bash
v4l2-ctl --device /dev/video0 --list-formats
```

Which yields something along the lines of this:
```bash
ioctl: VIDIOC_ENUM_FMT
	Type: Video Capture Multiplanar

	[0]: 'YUYV' (YUYV 4:2:2)
	[1]: '422P' (Planar YUV 4:2:2)
	[2]: 'NV16' (Y/CbCr 4:2:2)
	[3]: 'NV61' (Y/CrCb 4:2:2)
	[4]: 'YM61' (Planar YVU 4:2:2 (N-C))
	[5]: 'GREY' (8-bit Greyscale)
	[6]: 'NV21' (Y/CrCb 4:2:0)
	[7]: 'NV12' (Y/CbCr 4:2:0)
	[8]: 'NM21' (Y/CrCb 4:2:0 (N-C))
	[9]: 'NM12' (Y/CbCr 4:2:0 (N-C))
	[10]: 'YU12' (Planar YUV 4:2:0)
	[11]: 'YV12' (Planar YVU 4:2:0)
	[12]: 'RGGB' (8-bit Bayer RGRG/GBGB)
	[13]: 'GRBG' (8-bit Bayer GRGR/BGBG)
	[14]: 'GBRG' (8-bit Bayer GBGB/RGRG)
	[15]: 'BA81' (8-bit Bayer BGBG/GRGR)
	[16]: 'RG10' (10-bit Bayer RGRG/GBGB)
	[17]: 'BA10' (10-bit Bayer GRGR/BGBG)
	[18]: 'GB10' (10-bit Bayer GBGB/RGRG)
	[19]: 'BG10' (10-bit Bayer BGBG/GRGR)
	[20]: 'RG12' (12-bit Bayer RGRG/GBGB)
	[21]: 'BA12' (12-bit Bayer GRGR/BGBG)
	[22]: 'GB12' (12-bit Bayer GBGB/RGRG)
	[23]: 'BG12' (12-bit Bayer BGBG/GRGR)
```

### Getting a single frame from a video

For normal videos I found the tool [`kdenlive`](https://kdenlive.org/en/) to be quite helpful. For raw videos, I have started to use the [`vooya`](https://www.offminor.de/) tool, which sadly isn't open source but provides an easy way for viewing and raw videos and extracting frames from it. In order to get a `.png` from those raw frames, I used the [rawpixels.net](http://rawpixels.net/) tool, which additionally provides a few more format adjustment options.

### Observing the difference

Here is the scene captured directly with the `cam` application in an ordinary yuv format:
![Recorded with the cam application with the NV21 pixelformat](/images/bayer_order_analysis_cam.png)

I have used the following script for this job: [cam over ssh script](https://github.com/initBasti/NanoPC-T4_armbian_configuration/blob/main/libcam_from_nano.sh)

---

Here is what the raw images looked like when I recorded them with `v4l2-ctl` and converted to mp4 using `ffmpeg`.
![Observe the pattern with multiple flip configurations](/images/bayer_order_analysis_raw_mp4.png)

The command used for these images is:  
`ffmpeg -f rawvideo -vcodec rawvideo -video_size 2112x1568 -pixel_format bayer_bggr16le -i {input.raw} -qp 0 {output.mp4}`

---

And finally here are the raw images using the BGRA setting from *rawpixels.net*, I am really not happy with the resulting images as it is quite clear that the output format doesn't match the pixel-format, but these are the settings with the least amount errors during my tests:
![Recorded with v4l2-ctl, extracted a frame with vooya and adjusted the format with rawpixels.net](/images/bayer_order_analysis_rawpixels.png)
The settings for these images on [rawpixels.net](http://rawpixels.net/) are:
Predefined format: RGB555  
Pixel Format: BGRA  
Ignore Alpha: True Alpha First: False  
bpp1: 6  
bpp2: 4  
bpp3: 4  
bpp4: 2  
Litle Endian: True  
Pixel Plane: Packed  
alignment: 1  
subsamplig H: 1  
subsamplig V: 1  

Playing around with these settings is a great time sucker, I was currently not able to find a matching output format for the BG10 format, which is described [here](https://www.kernel.org/doc/html/latest/userspace-api/media/v4l/pixfmt-srggb10.html?highlight=bg10).

---

I would have loved to find a way to capture the Bayer mosaic directly but was unable to do so for the moment.

### Conclusion

In theory, I thought the order should change but looking at the images changed my mind. I have captured all images with the same pixel format and I cannot spot any corrupted areas.
