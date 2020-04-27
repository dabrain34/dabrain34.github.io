# How to detect a can using open technology

This tutorial aims to demonstrate that we can detect with various opensource technologies and
the openfoodfacts database a product and tell wether it's a can and get various details about it.

Indeed after some concerns about recycling and how could we manage to provide a simple solution
to avoir cans or bottles getting into the nature. I wanted to provide an easy solution for cities
to collect cans or any solid packaging according to the barcode or any code who could help to
identify it.

This project needs hardware and software materials, but in this blog post I aim to speak only about
software and how opensource can help to provide easy, reusable and *free* TM.

In this project, I will use a XPS9370 and its embedded camera and a regular ubuntu 16.04.

This solution will use for software:

  * Python 3.x
  * GStreamer 1.16
  * libzbar 0.23
  * OpenFoodFacts API
  * cerbero

First we gonna fetch the last branch of cerbero, Gstreamer buildsystem

```
# git clone git@gitlab.freedesktop.org:gstreamer/cerbero.git

# cd cerbero
```

Build `gst-plugins-bad` to provide zbar plugin
```
# ./cerbero-uninstalled bootstrap

# ./cerbero-uninstalled build gst-plugins-bad

# ./cerbero-uninstalled shell

```

```
 gst-launch-1.0 -m v4l2src device=/dev/video0 ! video/x-raw,width=640,height=480 ! videoconvert ! zbar ! xvimagesink
 ```

LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/lib/x86_64-linux-gnu/ ./gst-off-detector.py -x
