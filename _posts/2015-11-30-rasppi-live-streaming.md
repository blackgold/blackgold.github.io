---
layout: post
title: raspberry py live streaming
---

I was going on a month long vacation and I wanted to monitor my appartment. Given I have pan tilt camera with facial recognition, I thought of implementing a live feed, control camera and add motion detection to it. Its been working for two months and not even a glitch. I just need to implement motion detection though. 

1. checkout ffmpeg source and compile on raspberry pi. It will take overnight to compile, so run make and have a good nights sleep. You may get some missing library errors (libfaac-dev), you need to install them.

<snippet>
 git clone git://source.ffmpeg.org/ffmpeg.git
 cd ffmpeg/
 ./configure
 make 
 sudo make install
</snippet>

2. Following command captures the video and pipes the output to ffmpeg. Ffmpeg converts the source video to hls format and stores the fragments on disk and keeps updating the manifest file. ffmpeg also cleans up the old fragments

<snippet>
raspivid -n -ih -t 0 -ISO 800 -ex night -w 200 -h 100 -fps 25 -b 2000000 -o - | sudo ffmpeg -y  -i -  -c:v copy  -map 0  -f ssegment -segment_time 4 -segment_format mpegts -segment_list "/usr/share/nginx/www/live/stream.m3u8" -segment_list_size 10 -segment_wrap 20 -segment_list_flags +live -segment_list_type m3u8 "/usr/share/nginx/www/live/%03d.ts"
</snippet>

3. Configure apache server to serve fragments from the disk. 

3.1 Contents of pache config file.

<VirtualHost *:80>
        ServerAdmin webmaster@localhost

        DocumentRoot /usr/share/nginx/www/
        <Directory />
                Options FollowSymLinks
                AllowOverride None
        </Directory>
        <Directory /usr/share/nginx/www/>
                Options Indexes FollowSymLinks MultiViews
                AllowOverride None
                Order allow,deny
                allow from all
        </Directory>

        ScriptAlias /cgi-bin/ /usr/lib/cgi-bin/
        <Directory "/usr/lib/cgi-bin">
                AllowOverride None
                Options +ExecCGI -MultiViews +SymLinksIfOwnerMatch
                Order allow,deny
                Allow from all
        </Directory>

        ErrorLog ${APACHE_LOG_DIR}/error.log

        # Possible values include: debug, info, notice, warn, error, crit,
        # alert, emerg.
        LogLevel warn

        CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>

3.2 Following is the html index file

<html>
  <head>
    <title>PiVid</title>
  </head>
  <body>
    <video controls="controls" width="1280" height="720" autoplay="autoplay" >
      <source src="stream.m3u8" type="application/x-mpegURL" />
    </video>
  </body>
</html>

3.3 Cgi Scripts that increment the angle of servo. The script simply reads the current value from pipe and writes back the new angle.

<snippet>
#!/usr/bin/env python
# -*- coding: UTF-8 -*-

# enable debugging
import cgitb
cgitb.enable()

print "Content-Type: text/plain;charset=utf-8"
print

mpipe = open('/usr/share/nginx/www/live/pipe')
line = mpipe.readline()
angle = int(line)
newangle = angle + 10
print 'Got %d %d' % (angle, newangle )
mpipe.close()

mpipe = open('/usr/share/nginx/www/live/pipe','w')
mpipe.write(str(newangle))
mpipe.close()
</snippet>

3.4 Following code reads from the pipe and controls the servo. please refer to my servo repo for more info.

<snippet>
#include "servo.h"
#include <iostream>
#include <stdio.h>
#include <stdlib.h>
//#include <ncurses.h>
//#include <thread>
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>

const char *npipe = "/usr/share/nginx/www/live/pipe";

using namespace std;
int main () {
   char tmp[32];
   PCA9685 pwm;
   pwm.init(1,0x40);
   usleep(1000 * 100);
   printf ("Setting frequency..");
   pwm.setPWMFreq (61);
   usleep(1000 * 1000);

   while(true) {
      int fd = open(npipe, O_RDONLY) ;
      if ( fd < 0 ) {
         printf("Error opening pipe");
         return 0;
      }
      if( read(fd, tmp, sizeof(tmp)) > 0) { 
         printf("\n %d ", atoi(tmp));
      }
      int val = atoi(tmp);
      close(fd);
      if(val > 240 && val < 480) {
 	 pwm.setPWM(0,0,val);
      }  
      sleep(1);
   }
} 
</snippet>

4. Now modify the router to open port 80 to outside world. umm.

5. How to add motion detection to this?
Maybe redirect the feed to opencv program that can read frame by frame and compute the diff. And if there is diff send an alert email?

---
