---
layout: post
title: "ESExtractor: how to integrate a dependency-free library to the Khronos CTS"
date: 2023-04-19
canonical_url: "https://blogs.igalia.com/scerveau/esextractor-how-to-integrate-a-dependency-free-library-to-the-khronos-cts/"
---

# ESExtractor, how to integrate a dependency-free library to the Khronos CTS

Since the [Vulkan CTS](https://github.com/KhronosGroup/VK-GL-CTS) is now able to test and check [Vulkan Video support](https://www.khronos.org/news/press/vulkan-sdk-is-vulkan-video-ready)
 including video decoding, it was necessary to define the kind of media container to be used inside the test cases and the library
to extract the necessary encoded data.

In a first attempt, the [FFMpeg media toolkit](https://ffmpeg.org/) had been chosen to extract the video packets from the A/V ISO base media
format chosen as a container reference. This library was provided as a binary package and loaded dynamically at each
test run.

As Vulkan video aims to test only *video* contents, it was not necessary to choose a complex media container,
so first all the videos were converted to the elementary stream format for
H264 and H265 contents.
This is a very elementary format based on MPEG start codes and NAL unit identification.

To avoid an extra multimedia solution integrable only with binaries, a first attempt to replace FFmpeg was,
 to use GStreamer and an in-house helper library called [demuxeres](https://github.com/Igalia/GstVkVideoParser/tree/main/lib/demuxeres).
It was smaller but needed to be a binary still to avoid the glib/gstreamer system dependencies (self contained library).
it was a no-go still because the binary package would be awkward to support on the various platforms targetted by the the Khronos CTS.

So at [Igalia](https://www.igalia.com/), we decided to implement a minimal, dependency-free, custom library, written in C++
 to be compliant with the Khronos CTS and simple to integrate into any build system.

This library is called [ESExtractor](https://github.com/Igalia/ESExtractor)

## What is ESExtractor ?

ESExtractor aims to be a simple elementary stream extractor. For the first revision it was able to extract video data from
a file in the [NAL standard](https://en.wikipedia.org/wiki/Network_Abstraction_Layer).
The first official release was [0.2.4](https://github.com/Igalia/ESExtractor/releases/tag/release-v0.2.4). In this release,
only the NAL was supported with both the H264 and H265 streams supported.

As Vulkan Video aims to support more than H264 and H265 including format such as AV1 or VP9, the ESExtractor had to support multiple format.
A redesign has been started to support multiple format and is now available in the version 0.3.2.

## How ESExtractor works

A simple C interface is provided to maximise portability and use by other languages.

### es_extractor_new

```
ESExtractor extractor = es_extractor_new(filePath, "options"));
```

This interface returns the main object which will give you access to the packets according to the file path and the options given in the arguments.
This interface returns a ready to use object where the stream has been initially parsed to determine the kind of video during
the object creation.

Then you can check the video format with:

```
ESEVideoFormat eVideoFormat = es_extractor_video_format(extractor);
```

or the video codec with:

```
ESEVideoCodec eVideoCodec = es_extractor_video_codec(extractor);
```

It supports H264, H265, AV1 and VP9 for now.

### es_extractor_read_packet

This API is the main function to retrieve the available packets from the file. Each time this API is called,
the library will return the next available packet according to the format and the specific alignment (ie NAL) and
a status to let the application decide what to do next. The packet should be freed using `es_extractor_clear_packet`.


## Has CI powered by github

To test the library usage, we have implementing a testing framework in addition to a CI infrastructure
As github offers a very powerful worklow, we decided to use this platform to test the library on various architectures and platforms.
The [CI](https://github.com/Igalia/ESExtractor/actions) is now configured to release packages for 64 and 32 bits on Linux and Windows.


As usual, if you would like to learn more about Vulkan Video, ESExtractor or any other open multimedia framework, please contact [us](https://www.igalia.com/)!
