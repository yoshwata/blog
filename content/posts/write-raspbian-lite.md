---
title: "How to write Raspbian Stretch Lite"
date: 2019-05-01T21:45:44+09:00
draft: false
toc: false
images:
thumbnail: "/images/linktocat.jpg"
tags:
  - untagged
---

### Introduction

I introduce how to write Raspbian Stretch Lite image into your SD card.

What you need to do is these.

- HDMI display and the cable
- USB power adapter and the cable
- Raspberry Pi
- Micro SD card

I highly recommend that to write Raspbian Stretch Lite to your Raspberry Pi if you try arm CPU Kubernetes Cluster. That OS is well used by a lot of people who tried to build Kubernetes and there is a lot of information. Tools inside of Raspbian Stretch Lite is sufficient (like apt-get, curl ,vi), and also it launches in 20 or 30 secounds after power on. It was really helpful for person who likes to break the system.

This is a picture of my Raspberry Pi.
![raspberrypi](/images/raspberrypi.jpg)

It has a SD card slot on left side. You can launch any OS what you like with this small machine (but, the OS should be built for arm7l).

I use 32GB SD card because the price is reasonable. Smaller or larger card might be OK.

OK, let try to write OS image to your SD card.

### Download and Install Etcher

Please download Etcher from this site.  
https://www.balena.io/etcher/

You can select the platform of app so please select the appropriate one.

Then install and launch it. The window like below appears.

![etcher-select-image](/images/etcher-select-image.png)

### Download Raspbian Stretch Lite image

Go to this site.  
https://www.raspberrypi.org/downloads/raspbian/

Click the "Download ZIP" button of "Raspbian Stretch Lite", then download will start.

You are able to see file like below after unzip it.
```
2019-04-08-raspbian-stretch-lite.img
```

### Write image with Etcher

Please click "Select image" button and select the image what you get at privious step.
![etcher-select-image](/images/etcher-select-image.png)

Then, set the SD card to your PC. If it was recognized collectly, Etcher goes to next step.
![etcher-select-drive](/images/etcher-select-drive.png)

You can also select another SD card by "change button".
![](/images/etcher-change-drive.png)

Finally, click "Flash" button. The writing process starts and it finish after some minutes.
![](/images/etcher-flash.png)

### Log in to Raspbian

Set the SD card to your Raspberry Pi and HDMI to your display. You can see the launch process. After 20 or 30 seconds you can log in to the OS.

The default username and password is like below.
```
Username: pi
Password: raspberry
```

Please change the passward immidiately with `passwd` command for security.

