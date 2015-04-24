---
layout: post
title: Failed pan/tilt camera with facial detection.
---

Used raspberry pi to design this pan tilt camera. Raspberry can control pan/tilt smoothly.
Face detection works fairly well on its own.
When I run both the features symultaneously on threads the servo control goes haywired. The Servo goes into fits.
The pi is not able to skedule the thread controlling the servo's at exact 20 millisecond intervals.
Now its time to invest in a i2c servo controller.

![pan titl camera]({{ site.url }}/images/pantiltcamera.jpg)

---

