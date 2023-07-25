---
layout: post
title:  "Discover GStreamer full static mode"
date:   2023-07-25 12:10:45 +0200
categories: jekyll update
---

# How to embed statically your own tailored version of GStreamer in your application

Since the [gstreamer-full effort](https://dabrain34.github.io/2021/10/04/shrinking_gstreamer.html), it was possible to create a shared library which will embed the gstreamer framework in addition to its set of plugins.

Within this effort, it was also possible to register the selected plugins/features automatically by calling the `gst_init` method in your application linking with gstreamer-full.

This method was offering a gstreamer-full package with library, headers and pc files but it was not possible to embed gstreamer statically in your application and use it transparently.

# GstVkVideoParser: a standalone solution

In the journey to bring an open source solution for a video parser to the [Vulkan Conformance Test Suite](https://github.com/KhronosGroup/VK-GL-CTS), we chose first to use GStreamer as it was bringing all the parsing facilities
necessary to support the needed codecs such as H26x or VPx. This solution was supposed to be also cross platform and dragging as less as possible system dependencies.
Seen that GStreamer is usually dragging its own dependencies such as glib or orc and as we wanted to have a standalone [GstVkVideoParser](https://github.com/Igalia/GstVkVideoParser) library supported on Windows, a little bit
of work and love was necessary to add this to GStreamer.

Unfortunately this solution has not be retained by the Vulkan Video TSG not because it was not working but another parser has been made available and easy to integrate to the CTS at source level avoiding binary linkage, see Vulkan Video [change](https://github.com/KhronosGroup/VK-GL-CTS/commit/e5db10e7ae436dbd9b46dad9518f3254dd6eeea2).


# GStreamer as a full static library

With the `gstreamer-full` work, everything was almost ready to be used except to have `gstreamer-full` as a real static library and be able to link with it in any application.

Here is the [MR](https://gitlab.freedesktop.org/gstreamer/gstreamer/-/merge_requests/4128) merged and the challenges taken up:

## Adding gst-full-target-type=static

To generate the `gstreamer-full` dependency which will be statically linked into the application, we decided to introduce a new gst meson options, `gst-full-target-type`.

By default the `gstreamer-full` will be built as a *shared library* as before.

By passing `gst-full-target-type=static`, only static object will be generated and a package config file will be generated for gstreamer-full allowing the application to avoid to know what static library it needs to add the link line.
The GStreamer build system will take care of enabling/disabling the features/libraries you (do/don't) need.

## Initialize the plugins/features automatically

To avoid multiple call necessary to initialize GStreamer, it was also necessary to call the `gst_init_static_plugins` along with `gst_init` only in full-static mode but it was leading to a build issue.

Indeed most of tools/examples/tests are linking with `libgstreamer-1.0` which owns `gst_init ()` but to faciliate the plugins registration, it was necessary to move all the tools build after the `gstreamer-full` stage. A first [MR](https://gitlab.freedesktop.org/gstreamer/gstreamer/-/merge_requests/1581) has been performed to let gstreamer tools be built against gstreamer-full
but additional work was necessary for some core tools or helpers such as `gst-transcoder` or `gst-plugin-scanner` to avoid a linking issue.

## Disable tests and examples

In a future work all the tools/examples/tests should support the full-static mode but as GStreamer aims to be a shared object framework, we decided to leave this work for later and disable all the examples/tests in full static mode as most of the application using a tailor build won't need the examples and tests.

## Windows support

One of the goal of this work was to provide a Windows library to the Vulkan CTS free of dependency, which has been achieved but some additional work might be necessary to support
all of the use case, the GStreamer framework offer, especially on supporting library-dependent plugins.


## Give me an example ...

In the `GstVkVideoParser` [project](https://github.com/Igalia/GstVkVideoParser), various jobs are building Linux and Windows versions generating a library without any GStreamer/glib dependencie, everything is embedded inside the library, as you can see in this [GitHub' Actions](https://github.com/Igalia/GstVkVideoParser/actions).

In this project, GStreamer is used as a meson subproject/wrap which allows to build GStreamer along of GstVkVideoParser. This can be possible easily by adding the following file to your meson project

subprojects/gstreamer-1.0.wrap
```
[wrap-git]
directory=gstreamer-1.0
url=https://gitlab.freedesktop.org/gstreamer/gstreamer.git
revision=main

[provide]
dependency_names = gstreamer-1.0, gstreamer-base-1.0, gstreamer-video-1.0, gstreamer-audio-1.0
```

and then add the following lines to your `meson.build` to depend on `gstreamer-full`

meson.build
```
gstreamer_full_dep = dependency('gstreamer-full-1.0', fallback: ['gstreamer-1.0'], required :true)
```

In order to build a project, library or application which is using a tailored version of GStreamer you can follow this [configure example](https://github.com/Igalia/GstVkVideoParser/blob/main/configure_gst_full.py):

```
$ meson buildfull-static --default-library=static --force-fallback-for=gstreamer-1.0,glib,libffi,pcre2 -Dauto_features=disabled -Dglib:tests=false -Djson-glib:tests=false -Dpcre2:test=false -Dvkparser_standalone=enabled -Dgstreamer-1.0:libav=disabled -Dgstreamer-1.0:ugly=disabled -Dgstreamer-1.0:ges=disabled -Dgstreamer-1.0:devtools=disabled -Dgstreamer-1.0:default_library=static -Dgstreamer-1.0:rtsp_server=disabled -Dgstreamer-1.0:gst-full-target-type=static_library -Dgstreamer-1.0:gst-full-libraries=gstreamer-video-1.0, gstreamer-audio-1.0, gstreamer-app-1.0, gstreamer-codecparsers-1.0 -Dgst-plugins-base:playback=enabled -Dgst-plugins-base:app=enabled -Dgst-plugins-bad:videoparsers=enabled -Dgst-plugins-base:typefind=enabled
```

In this case we are disabling everything in GStreamer by using `-Dauto_features=disabled` and some enabled features such as `ges`, `libav` etc. and enable only what we need as plugins, `playback`, `app`, `videoparsers` and `typefind`.

And then we are enabling the static build with `--default-library=static` and `-Dgstreamer-1.0:gst-full-target-type=static_library`.

## Next ...

As you can see, it's quite easy now to build an application and depends on `gstreamer-full` static build, but there is still some issues to address such as the plugins dependency which might be not static and some other platform specific issue such as the `gstreamer-full` symbols export on Windows.

You can follow some open issues such as:

https://gitlab.freedesktop.org/gstreamer/gstreamer/-/issues/1629
https://gitlab.freedesktop.org/gstreamer/gstreamer/-/merge_requests/2104


As usual, if you would like to learn more about Vulkan Video, GStreamer or any other open multimedia framework, please contact [us](https://www.igalia.com/)!
