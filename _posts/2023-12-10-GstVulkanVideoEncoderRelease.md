---
layout: post
title:  "Vulkan Video encoder in GStreamer"
date:   2023-12-10
categories: jekyll update
---

# Vulkan Video encoder in GStreamer

During the last months of 2023, we, at [Igalia](https://www.igalia.com/about/), decided to focus on the latest provisional specs proposed
by the [Vulkan Video Khronos TSG group](https://www.khronos.org/news/press/vulkan-sdk-is-vulkan-video-ready) to support encode operations in an open reference.

As the [Khronos TSG Finalizes Vulkan Video Extensions for Accelerated H.264 and H.265 Encode](https://www.khronos.org/blog/khronos-finalizes-vulkan-video-extensions-for-accelerated-h.264-and-h.265-encode?mc_cid=e4afdbcd22&mc_eid=47d3c1b7bb)
the 19th of December 2023, here is an update on our work to support both h264 and h265 codec in GStreamer.

This work has been possible thanks to the relentless work of the Khronos TSG during the past years, including the IHV vendors, such as
AMD, NVIDIA, Intel, but also Rastergrid and its massive work on the specifications wording and the vulkan tooling, and of course Cognizant for its valuable
contributions to the [Vulkan CTS](https://github.com/KhronosGroup/VK-GL-CTS).

This work started with moving structures in drivers and specifications along the CTS tests definition. It has been a joint venture
to validate the specifications and check that the drivers suppports properly the features necessary to encode a video
with h264 **and** h265 codecs. *Breaking news, more formats should come very soon...*

## The GStreamer code

### Building a community

Thanks first to [Lynne](https://lynne.ee/pages/about.html) and [David Airlie](https://airlied.blogspot.com/), on their RADV/FFmpeg effort,
a first implementation was available last year to validate the provisional specifications of the encoders.
This work helped us a lot to design the GStreamer version and offer a performant solution for it.

The GStreamer Vulkan bits have been also possible thanks to the tight collaboration with the CTS folks which helped us a lot to understand the
crashes we could experience without any clues in the drivers.

To write this GStreamer code, we got inspiration from the code written by the GStreamer community along the [GstVA encoders](https://gstreamer.freedesktop.org/documentation/va/index.html?gi-language=c) and
v4l2codecs plugins. So great kudos to the GStreamer community and especially to He Junayan from Intel, Collabora folks paving the way to stateless codecs and of course
Igalia's past contributions.

And finally the active and accurate reviews from Centricular folks to achieve a valid synchronized operation in Vulkan,
see for example <https://gitlab.freedesktop.org/gstreamer/gstreamer/-/merge_requests/5079> in addition to their initial work to
support Vulkan.


### Give me the (de)coded bits

The code is available on the [GStreamer MR](https://gitlab.freedesktop.org/gstreamer/gstreamer/-/merge_requests/5739) and we plan to have it ready in the 1.24 version.

This code needs at least the Vulkan SDK version 1.3.274 with KHR Vulkan Video encoding extension support. It should be available publicly when the specifications and the tooling will be available
along the next SDK release early 2024.

To build it, nothing is more simple than (after installing the [LunarG SDK](https://vulkan.lunarg.com/#new_tab) and the [GST requirements](https://gstreamer.freedesktop.org/documentation/installing/building-from-source-using-meson.html?gi-language=c)):

```
$ meson setup builddir -Dgst-plugins-bad:vulkan-video=enabled
$ ninja -C builddir
```

And to run it, you'll need an IHV driver supporting the KHR extension, which should happen very soon, or the community driven driver such as [Igalia's Intel ANV driver](https://gitlab.freedesktop.org/zzoon/mesa/-/tree/h264enc_anv_4?ref_type=heads).
You can also try the AMD RADV driver available [here](https://gitlab.freedesktop.org/airlied/mesa/-/commits/radv-vulkan-video-encode-h2645-spec-latest/?ref_type=undefined)
when KHR will be available.

To encode and validate the bitstream you can run this transcode GStreamer pipeline:

```
$ gst-launch-1.0 videotestsrc ! vulkanupload ! vulkanh265enc ! h265parse ! avdec_h265 ! autovideosink
```

As you can see a `vulkanupload` is necessary to upload the GStreamer buffers to the GPU memory, then the buffer will be encoded
through the `vulkanh265enc` which will generate a byte-stream AU aligned stream which can be decoded with any decoder, here the FFMpeg one.

We are planning to simplify this pipeline and allow to select an encoder from a multiple device configuration.

It supports most of the Vulkan video encoder features and can encode I, P and B Frames for better streaming performance. See this
[blog post](https://www.rastergrid.com/blog/multimedia/2021/05/video-compression-basics/)

## Challenges

One of the most important challenge here was to put **all** the bits together and understand when the drivers tell you that you are doing
something wrong with only an obscure segfault.

The vulkan process to encode *stateless* a raw bitstream is quite simple as it will be:

 - Initialize the video session with the correct video profile (h264,h265 etc...)
 - Initialize the sessions paramaters such as SPS/PPS which will be used by the driver during the encode session
 - Reset the codec state
 - Encode the buffers:
    - Change the quality/rate control if necessary
    - Retrieve the video session parameters from the driver if necessary
    - Set the bitstream standard parameters such as slice header data.
    - Set the begin video info to give the context to the encoder for the reference
    - Encode the buffer
    - Query the encoded result
    - End the video coding operation
    - *repeat ..*


The challenge here was not the state machine design but more the understanding of the underlying bits necessary to tell
the driver what and how to encode it. Indeed to encode a video stream, a lot of parameters (more than 200..) are necessary and a bad
use of these parameters can lead to obscure crashes in the driver stack.

So first it was important to validate the data provided to the Vulkan drivers through the Validation layer. This part
in implementing an application using Vulkan is the first and essential step to avoid headaches, see [here](https://vulkan-tutorial.com/Drawing_a_triangle/Setup/Validation_layers)

You can checkout the code from [here](https://github.com/KhronosGroup/Vulkan-ValidationLayers) to get the latest one and configure it to validate
your project. Otherwise it should come from the Vulkan SDK.

This step helped us a lot to understand the mechanics expected by the driver to provide the frames and their references
to encode those properly. It also helped to use some API such as the rate control or the quality level.

But Validation Layers cant do everything and that's why designing, reviewing a solid codebase in CTS as a reference
is crucial to implement the GStreamer application or any other 3rd party application. So a lot of time in Igalia has been
dedicated to ease and simplify the CTS tests to facilitate its usage.

Indeed the Validation Layers do not validate the Video Standard parameters for example such as the one provided for the format
underlying specific data.

So we spent a few hours checking some missing parameters in CTS as in GStreamer which was leading to obscure crash, especially
when we were playing with references. A blog post should follow explaining this part.

To debug this missing bits, we used some facilitating tools such as [GfxReconstruct Vulkan layer](https://vulkan.lunarg.com/doc/view/latest/linux/capture_tools.html)
 or the `VK_LAYER_LUNARG_api_dump` which allows you to get an essential dump of your calls and compare the results with other implemetations.

## Conclusion

It was a great journey providing a cross platform solution to the community to encode video using largely adopted codec
such as h264 and h265. As said before other codecs will come very soon, so stay tuned!

I mentioned the team work in this journey as without the dedication of all, it wouldn't have been possible to achieve this work
in a short time span.

And last but not least, I'd like to special thank also Valve to sponsor this work without whom it wouldn't have been possible to go as far and as fast!

We'll attend the [Vulkanised 2024](https://www.khronos.org/events/vulkanised-2024) event where we'll demonstrate this work.
Feel free to come and talk to us there. We'll be delighted to hear and discuss your doubts or brilliant ideas !

As usual, if you would like to learn more about Vulkan, GStreamer or any other open multimedia framework, please contact [us](https://www.igalia.com/)!
