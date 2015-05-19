---
layout: post
title: pan/tilt camera with facia recognition (attempt 1) 
---

Used i2c servo controller to control the pan tilt servo's. Servo's work accurately now and servo controller takes responsibility of maintaining the pulse.
Disabled facial recognition for this excercise and simply used facial detection. Used opencv's haarcascade_frontalface_alt.xml for this purpose.
Now that I can draw a window around the faces, All I need to do is follow the face. As the face moves the coordinated of the rectangle shift.
Keep moving the servo's to ensure the rectangle is always in facus. 
Checkout my public repos for servo controller code. Will publish facial detection program soon. 

As simple as the idea seems to be, there are various problems to be solved
1. I am only able to do 3.4 frames per second. Need to investigate this and optimize the code to speed it up.
2. Ensure the camera is firm without much vibration. Same goes with servos.  Need a solution to keep the frame sturdy.
3. Cannot detect small faces. Like when people are far from camera. No zoom in and zoom out feature.
4. Need to setup a server that streams the camera feed. Xterm display is choppy.

[![Experiment Video](http://img.youtube.com/vi/6wD5w_dazbs/0.jpg)](http://www.youtube.com/watch?v=6wD5w_dazbs)

---

