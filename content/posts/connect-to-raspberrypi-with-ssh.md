---
title: "Connect to Raspberrypi with SSH"
date: 2019-05-03T17:48:20+09:00
draft: false
toc: false
images:
tags:
  - raspberrypi
---

### Introduction

After you setup Raspberry pi like [this page](https://yoshwata-blog.netlify.com/posts/write-raspbian-lite/), you are able to execute any command and get response from HDMI monitor. To make it more useful, you might think to log in from remote host. I will introduce the way how to set up ssh with Raspberry pi.

### Set password of pi account

First of all you should change the password of pi account. Please hit this command.

```bash
$ passwd
```
After hit this command, a interactive setting will start, so please follow the guidance.

### Set SSH on from raspi-config

Please execute this command.
```bash
$ sudo raspi-config
```

Then this screen will appear.

Go to `5 Interfacing Options` -> `P2 SSH`

And set `yes`.

### (optional) Set SSH from cli

When I researching the way to set the SSH setting from command, I found interesting way to do this.

Command is this.
```
$ raspi-config nonint do_ssh 0
```
It will also enable SSH

If you have other settings that you want to set from command, please this code might be useful.  
https://github.com/raspberrypi-ui/rc_gui/blob/master/src/rc_gui.c

 

