---
layout: post
title:  "Connect HomeKit lights"
date:   2023-12-26 09:49:29 +1200
category: software
tags: mac ios homekit smart wifi network apple
---

I have a number of "smart lights" at home from 3 different vendors.
I recently had to change to a new iPhone, and after restoring from a backup,
some of my carefully setup lights stopped working in the Apple Home (HomeKit) application. The funny thing was the lights were still working with their
vendor-specific apps.

My attempts to persuade the vendor-specific apps to reinstall the light with
HomeKit were fruiteless. I tried to re-add the lights to the Home app, and
despite the process starting well (the ligth was detected by the Home app), then
after a long wait I was getting an error saying that the connection had failed.

I tried reseting the lights, uninstalling the vendor-specific apps, etc. Nothing
was working.

It took me a while to figure out that the problem was the wifi. These devices are
compatible only with 2.4GHz wifi. I have a Synology router which supports both
5GHz and 2.4GHz. Until some time ago, it was announcing those two frequencies as
different wifis, each with its own SSID. But recently (did I change that?) it
started bundling both wifis under a single SSID. I went to the Synology router
admin screens, found the setting to "unbundle" the two wifis, and then switched my
phone to the 2.4GHz wifi, which finally allowed me to complete the setup!
