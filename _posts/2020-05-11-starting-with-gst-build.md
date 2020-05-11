---
layout: post
title: "Starting with gst-build"
date: 2020-05-11
---

## How to add a new log in GStreamer using gst-build


### The State of the Art:

GStreamer relies on multiple repositories such as base and good to build its ecosystem, and now owns more than 30 projects in Gitlab. So, a unified tool/build system has always been necessary to build a specified version.

For over a decade, a script named `gst-uninstalled` was present in the `gstreamer/scripts` directory to build the whole solution. Although this tool was not very flexible and was missing some options in the command line, it was good enough if you wanted to tackle a surprising bug in our favorite framework. But it was not as good at providing a real swiss-army knife approach to build GStreamer and its dependencies.

Another build system called [Cerbero](https://gitlab.freedesktop.org/gstreamer/cerbero), created a few years ago, provides a standalone solution to build GStreamer packages. This solution offers a wide range of options in addition to a proper sandbox to avoid system dependencies and to be able to prepare packages that include third party software dependencies for a given version. Cerbero is written in Python and can create builds for the host machine like gst-uninstalled but also for various common targets depending on the host. Indeed a Linux regular desktop host will be capable to cross-build GStreamer for x86(32/64bits) but also for architecture such ARM and system such as Microsoft Windows. It can also create builds for Android and iOS. Despite a shell environment allowing artifacts testing, Cerbero is not really convenient for a day to day development related to GStreamer as a new plugin development or a bug fix as it is not easy to update to the last revision without loosing a current work, or to test another branch of GStreamer


### The Rise of gst-build:

In order to improve this situation,  [gst-build](https://gitlab.freedesktop.org/gstreamer/gst-build) was born.
Taking advantage of the flexibility of the rising build system, [meson](https://mesonbuild.com/), gst-build has been implemented to replace gst-uninstalled and provide a quick and smooth environment to hack into GStreamer and its dependencies.


### Autotools Is Dead, Long Live Meson:

Since GStreamer 1.18, meson has been chosen as the only build system for the official GStreamer repositories. For its simplicity, speed and flexibility, meson replaced autotools, so it is also perfect to use with gst-build. Indeed gst-build is just a meson project including GStreamer sub-projects with options to enable/disable selected sub-projects.

So lets take a look on how to get started with gst-build:


### A first step using gst-build:

#### A few 'bits' about it:

gst-build is mainly a `meson.build` project. It reads .wrap files which are located in the subprojects folder to determine the elements of the project such as GStreamer or gst-plugins-base. These subprojects use the meson build system as well. gst-build comes with the essential projects you need to start using GStreamer and build it almost without system dependencies. gst-build bundles libffi or glib in the `subprojects` directory. It can also gather dependencies using pkg-config from the system to build the GStreamer plugins such as flac, for example, which needs libflac to build.

#### Environment

As we have to choose a real development environment, a 64 bit machine has been selected:

 * Ubuntu 18.04
 * Bash Shell

#### Prerequisites

 * build-essential (gcc)
 * python3
 * git
 * meson
 * ninja

#### Install meson and ninja

Here are the essential dependencies you need to install before running meson and ninja.

```
$ sudo apt install build-essential python3 git ninja python3-pip
```

You can now install meson from the `pip` repository.

```
$ pip3 install --user meson
```

This will install `meson` into `~/.local/bin` which may or may not be included automatically in your PATH by default.


#### Fetch and Configure

This step will download the GStreamer repositories including some dependencies such as glib etc. into the `subprojects` folder. Basically it tries to download as many `mesonified` third party libraries as possible,  and breaking news the `cmake` ones, as a bridge has been implemented recently if necessary.

```
$ git clone https://gitlab.freedesktop.org/gstreamer/gst-build
$ cd gst-build
$ meson build --buildtype=debug
```

```
...

All GStreamer modules 1.17.0.1

  Subprojects
                        FFmpeg: YES
                         dssim: YES
                    gl-headers: YES
                      graphene: YES
                  gst-devtools: YES
          gst-editing-services: YES
                  gst-examples: YES
    gst-integration-testsuites: YES
                     gst-libav: YES
                       gst-omx: YES
               gst-plugins-bad: YES
              gst-plugins-base: YES
              gst-plugins-good: YES
                gst-plugins-rs: NO
              gst-plugins-ugly: YES
                    gst-python: NO
               gst-rtsp-server: YES
                     gstreamer: YES
               gstreamer-sharp: Feature 'sharp' disabled
               gstreamer-vaapi: YES
                         gtest: NO
                   libmicrodns: YES
                       libnice: YES
                        libpsl: YES
                       libsoup: NO
                      openh264: YES
                           orc: YES
                     pygobject: NO
                        sqlite: YES
                      tinyalsa: NO
                          x264: YES
Option buildtype is: debug [default: debugoptimized]
Found ninja-1.8.2 at /usr/bin/ninja

```

After this step, a newly created folder named `build` should be ready to be used by `ninja` to build the binaries.

As you may notice, `--buildtype=debug` has been added to the command line to get a fully debugable result without optimization. I invite you to this [page](https://mesonbuild.com/Builtin-options.html#core-options) if you want to fine-tune the build.

##### Build gst-build

This step will build all GStreamer libraries in addition to the plugins from base/good/bad/ugly/libav if their dependencies have been met or built by `gst-build` (ie glib, openh264 etc.).

```
$ ninja -C build
```


### Test gst-build

This command will create an environment where all tools and plugins built previously are available in the environment as a superset of the system environment with the right environment variables set.

```
$ ninja -C build devenv
```

A prefix to your prompt should be shown as


```
[gst-master] bash-prompt $
```

```
[gst-master] bash-prompt $ env | grep GST_
```

From this environment you are now ready to use the power of GStreamer, and even implement new features in it without the fear of using out of date version.

From this shell, you are also able to compile without exiting the environment except when a configure step is necessary. This feature is very convenient to test a branch or fix a bug. Go to the `subprojects` folder and modify the code directly and then call `ninja -C ../../build`.

```
[gst-master] bash-prompt $ gst-inspect-1.0
```

#### Let's add a log line in gst-plugins-base

In this tutorial, I will explain how to add a log line in videotestsrc element, gst-plugins-base's plugin, rebuild using gst-build and test that the new log is now displayed.

##### Edit the file

```
vim subprojects/gst-plugins-base/gst/videotestsrc/gstvideotestsrc.c
```
Go to the method `gst_video_test_src_start` and add the line:

```
GST_ERROR_OBJECT (src, ""Starting to debug videotestsrc, is there an error ?");
```
This will add a runtime log with the ERROR level. For more information about debugging facilities in GStreamer, visit the [following page](https://gstreamer.freedesktop.org/documentation/tutorials/basic/debugging-tools.html?gi-language=c)

Then close the editor.

##### Build with gst-build

```
$ ninja -C build
```

You should see that only the file `gstvideotestsrc.c` rebuilt.

##### Test the changes

In order to enable the logs, you have to export the environment variable `GST_DEBUG`.

Let's start the playback and display the result in the terminal. The following command will display all the log from videotestsrc with the category ERROR(1).

```
GST_DEBUG=videotestsrc:1 gst-launch-1.0 videotestsrc num-buffers=1 ! fakevideosink
```

You should have this output:

```
Setting pipeline to PAUSED ...
0:00:00.225273663 21743 0x565528ab7100 ERROR           videotestsrc gstvideotestsrc.c:1216:gst_video_test_src_start:<videotestsrc0> Starting to debug videotestsrc, is there an error ?
Pipeline is PREROLLING ...
Pipeline is PREROLLED ...
Setting pipeline to PLAYING ...
New clock: GstSystemClock
Got EOS from element "pipeline0".
Execution ended after 0:00:00.033464391
Setting pipeline to PAUSED ...
Setting pipeline to READY ...
Setting pipeline to NULL ...
Freeing pipeline ..
```

##### Update gst-build

This command will update all the repositories and will reissue a build.
```
$ ninja -C build_dir update
```

#### Adding a new repository

Better to be outside of `devenv` env.
If you want to add a new repository and work in this environment. Very simple and handy way, you'll have to:

```
$ cd subprojects
$ git clone my_subproject
$ cd ../build
$ rm -rf * && meson .. -Dcustom_subprojects=my_subproject
```

And then you can go in your `subproject`, edit, change, remove even stare at his beauty.

```
$ ninja -C ../../build
$ ninja -C ../../build uninstalled
```


If you would like to learn more about gst-build or any other parts of GStreamer, please [contact us](https://www.collabora.com/contact-us.html)
