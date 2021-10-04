---
layout: post
title: "Generate a minimal GStreamer build, tailored to your needs"
date: 2021-10-04
---

GStreamer is a powerful multimedia framework with over 30 libraries and more than 1600 elements in 230 plugins providing a wide variety of functionality. This makes it possible to build a huge variety of applications, however it also makes it tricky to ship in a constrained device. Luckily, most applications only use a subset of this functionality, and up until now there wasn't an easy way to generate a build with just enough GStreamer for a specific application.

Thanks to a partnership with Huawei, you can now use [gst-build] to generate a minimal GStreamer build, tailored to a specific application, or set of applications. In this blog post, we'll look at the major changes that have been introduced in GStreamer to make this possible, and provide a small example of what can be achieved with minimal, custom builds.

## gst-build

[gst-build] is the build system that GStreamer developers use. In previous posts,
we described how to get started on [Linux](https://www.collabora.com/news-and-blog/blog/2020/03/19/getting-started-with-gstreamer-gst-build/)
or [Windows](https://www.collabora.com/news-and-blog/blog/2019/11/26/gstreamer-windows/) and how to use it as a daily development tool.

Since GStreamer 1.18, it is possible to build all of GStreamer into a single shared library called [gstreamer-full]. This library can include not only GStreamer's numerous libraries, but also all the plugins and other GStreamer dependencies such as a GLib. Applications can then either dynamically or statically link with gstreamer-full.

### Creating the gstreamer-full combined library

By providing `-Ddefault_library=static -Dintrospection=disabled` to the Meson configure command line, it will generate a static build 
of all the GStreamer libraries which support the static scheme. This will also produce a shared library called [gstreamer-full] containing all of GStreamer.
For now the GObject introspection needs to be disabled as the static build is not ready to support it (see [gst-build-167]).

### Tailoring GStreamer

Generating a combined library doesn't by itself reduce the total size. To achieve this goal, we need to select which libraries and plugins are included. 

[gst-build] is a highly configurable build system that already provides options to select which plugins are built. But using the [gstreamer-full] mechanism, one can select exactly which libraries are included in the final [gstreamer-full] library by passing the `-Dgst-full-libraries=` argument to meson. The plugins are then automatically included according to the configuration and the dependencies available.

Lets have an example:

```
$ meson build-gst-full \
  --buildtype=release \
  --strip \
  --default-library=static \
  --wrap-mode=forcefallback \
  -Dauto_features=disabled \
  -Dgst-full-libraries=app,video,player \
  -Dbase=enabled \
  -Dgood=enabled \
  -Dbad=enabled \
  -Dgst-plugins-base:typefind=enabled \
  -Dgst-plugins-base:app=enabled \
  -Dgst-plugins-base:playback=enabled \
  -Dgst-plugins-base:volume=enabled \
  -Dgst-plugins-base:videoconvert=enabled \
  -Dgst-plugins-base:audioconvert=enabled \
  -Dgst-plugins-good:audioparsers=enabled \
  -Dgst-plugins-good:isomp4=enabled \
  -Dgst-plugins-good:deinterlace=enabled \
  -Dgst-plugins-good:audiofx=enabled \
  -Dgst-plugins-bad:videoparsers=enabled
```

In this example, we generate a [gstreamer-full] library using only the feature we explicitly specify. The first step to do that is to disable the automatic selection of features based on which dependencies are already installed (`-Dauto_features=disabled`). Then we explictly enable the features of each subpackage that we want (ie `-Dgst-plugins-base:typefind=enabled -Dgst-plugins-base:app=enabled`) and we use `-Dgst-full-libraries=app,video,player` to tell [gst-build] to bundle and expose only those specific libraries (app, video, player) in [gstreamer-full]. We also force meson to build as many dependencies itself by using the *fallback* with (`--wrap-mode=forcefallback`), this way, those dependencies will be included in the [gstreamer-full] library.

### Tailoring it further

In our collaboration with Huawei, we decided to push further the idea of tailoring GStreamer to increase the granuality beyond
the plugin to be able to select individual element or other features inside each plugin.

GStreamer plugins contain different kind of features. The most common type of *plugin features* are *elements*, but you can select other type of loadable features such as [device provider], [typefind] and [dynamic-type].
For example, the ALSA plugin includes a device provider and various elements such as `alsasrc` and `alsasink`.

One key goal of the project was to be able to build only the features needed, reducing the binary file size. For example if the user selects only one element such as `flacparse` from the `audioparsers` plugin, the code used by the other parsers should not be included in the final binary.

Note that for now, this project has been focused on Linux platforms and has not been tested on other platforms such as Windows or macOS.

We first experimented with the linker option to garbage collect sections (`--gc-sections`). This option removes code sections which are not
used by any part of the final program, except the public library methods.  With the standard compiler options, there is normally a section per C source file, but using the  `-ffunction-sections` and `-fdata-sections` compiler options, the compiler will generate one section per function and per data symbol.

But according to the documention, this feature must be used with care as it can bring an overhead of code 
if no sections can be garbage-collected and also because of incompatibility
with debugger and slowness ( See `-fdata-sections` comment in [gcc man page](https://linux.die.net/man/1/gcc)).

As GStreamer is a very widely used project, we decided to avoid this solution as it could possibly lead to inconsistent results.

### Splitting the code inside GStreamer

Instead of creating new sections automatically, we decided to play with linker rules. The linker (ld) already only pulls in object files if they are called
by other objects. So this has the effect of entirely omitting any code that isn't used by the current program.

Up until now, every plugin had a function that would call the registration function for each feature present in the plugin. This function is called when
the plugin is loaded. This plugin initialization function was the only one in each plugin with a predictable name. To be able to select plugins, we needed to expose a registration function for each feature. We were very careful to put these in the same file where the feature is implemented.

The plugin initialization function is now in its own file that can be ignored when linking features one by one. This work required modifying every single GStreamer plugin.
A registration method has been declared through a set of macro for each of feature available in GStreamer official repositories.

To declare an element, you should use the macro `GST_ELEMENT_REGISTER_DEFINE(element, "name", TANK, GST_TYPE)`. From any plugin you can register the element
calling `GST_ELEMENT_REGISTER(element, plugin)`.

The final part of this work was to create a "static" plugin in [gstreamer-full] which will contain all the features (element etc.)
selected by the [gst-build] configure with `-Dgst-full-elements=`.

With these in place, all the features which are not selected don't get included.


### Compose your GStreamer feature(s) menu:

Five new options have been added to gst-build:

 - gst-full-plugins: Select the plugin you'd like to include. By default, this is all the plugins enabled by the build process. At least one must be passed or all will be included.
 - gst-full-elements: Select the element(s) using the `plugin1:elt1,elt2;plugin2:elt1` format
 - gst-full-typefind-functions: Select the typefind(s) using the `plugin1:tf1,tf2;plugin2:tf1` format
 - gst-full-device-providers: Select the decide-provider(s) using the `plugin1:dp1,;dp2;plugin2:dp1` format
 - gst-full-dynamic-types: Select the dynamic-type(s) using the `plugin1:dt1,;dt2;plugin2:dt1` format

You can find more information about the work achieved in the [gst-build-199] merge request.

#### A light *menu*

Let's start with a default build with all features and get some metrics to see the benefits.
The [gstreamer-full] library will embed as much as possible its external dependencies
such as `glib`. Plugin dependencies will be kept dynamic but as soon as we'll select one or another
plugin/feature, the dependency will be removed.

The build has been performed using an official GStreamer Fedora Docker image.

```
$ docker pull registry.freedesktop.org/gstreamer/gst-plugins-bad/amd64/fedora:2021-03-30.0-master
```

```
$ meson build-gst-full \
  -Ddefault_library=static \
  -Dintrospection=disabled \
  --buildtype=release \
  --strip \
  --wrap-mode=forcefallback

$ ninja -C build-gst-full
```

After a successful build, we can reconfigure the last [gstreamer-full] by providing new options to the meson comamnd line through `--reconfigure`.
In this use-case, we'll enable only 3 elements from the coreelements plugin in GStreamer.

```
$ meson build-gst-full --reconfigure -Dgst-full-plugins=coreelements '-Dgst-full-elements=coreelements:filesrc,fakesink,identity' '-Dgst-full-libraries=[]'
$ ninja -C build-gst-full
```
In this example we are first passing the plugin(s) you'd like to enable, `coreelements` and then passing the elements we'd like to include `filesrc`, etc. We remove all additional GST libraries except the core library.


|lib (stripped)| default | tailored |
|--|--|--|
|ligstreamer-full.so|49208656 (49.2 M)|3250256 (3.2M)|

This library can now be used through its pkg-config as [gstreamer-full] file to build a custom GST application.

#### Use of a linker script

As a final touch, we have also added an option to provide gst-build with a linker script to select exactly what gets included in the final [gstreamer-full] library. With this linker script you are now able to drop all the public code which is not used by your application
and keep only the necessary code. See [gst-build-195]

The option is:  `gst-full-version-script=path_to_version_script`

# Wrapping up

Some interesting merge requests:

- [gst-build-199]
- [gstreamer-661]
- [gst-plugins-base-1029]
- [gst-plugins-good-876]
- [gst-plugins-bad-2038]
- [gst-plugins-bad-2110]
- [gst-plugins-bad-2116]
- [gst-plugins-ugly-79]

All this work is now available upstream (1.19.0) and should available in the next 1.20 release of GStreamer.

As usual, if you would like to learn more about meson, gst-build or any other parts of GStreamer, please contact us!

[gst-build]: https://gitlab.freedesktop.org/gstreamer/gst-build
[gstreamer-full]: https://gitlab.freedesktop.org/gstreamer/gst-build#static-build
[gst-build-167]: https://gitlab.freedesktop.org/gstreamer/gst-build/-/merge_requests/167
[gst-build-195]: https://gitlab.freedesktop.org/gstreamer/gst-build/-/merge_requests/195
[gst-build-199]: https://gitlab.freedesktop.org/gstreamer/gst-build/-/merge_requests/199
[gst-build-207]: https://gitlab.freedesktop.org/gstreamer/gst-build/-/merge_requests/207/
[gstreamer-661]: https://gitlab.freedesktop.org/gstreamer/gstreamer/-/merge_requests/661
[gst-plugins-base-1029]: https://gitlab.freedesktop.org/gstreamer/gst-plugins-base/-/merge_requests/1029
[gst-plugins-good-876]: https://gitlab.freedesktop.org/gstreamer/gst-plugins-good/-/merge_requests/876
[gst-plugins-bad-2038]: https://gitlab.freedesktop.org/gstreamer/gst-plugins-bad/-/merge_requests/2038
[gst-plugins-bad-2110]: https://gitlab.freedesktop.org/gstreamer/gst-plugins-bad/-/merge_requests/2110
[gst-plugins-bad-2116]: https://gitlab.freedesktop.org/gstreamer/gst-plugins-bad/-/merge_requests/2116
[gst-plugins-ugly-79]: https://gitlab.freedesktop.org/gstreamer/gst-plugins-ugly/-/merge_requests/79
[device provider]: https://gstreamer.freedesktop.org/documentation/gstreamer/gstdeviceprovider.html#gstdeviceprovider-page
[typefind]: https://gstreamer.freedesktop.org/documentation/gstreamer/gsttypefindfactory.html?gi-language=c#GstTypeFindFactory
[dynamic-type]: https://gstreamer.freedesktop.org/documentation/gstreamer/gstdynamictypefactory.html?gi-language=c
