# RaspPi YouTube Live Stream
We have an outdoor fountain whose major purpose is to serve the birds that enjoy taking baths in it.  

I wanted to view the birds at our fountain from anywhere on a Smart TV, tablet, phone, or PC.
## Streaming to A Smart TV
I used a Zero W Raspberry Pi and the Raspberry Pi camera to stream video to a YouTube Live Stream channel.

I learned a lot given I was clueless on how to do this.  Please let me know how this can be done better.

- TODO: Audio
To simplify the build, the stream does not include audio.  My goal is to add this important feature once the webcam works awhile.

Let our adventure begin!
# YouTube Live Stuff
This section discusses what we need to do prior to our Raspberry Pi sending audio/video to a YouTube Live Streaming channel.
## Get a Channel
We need a live stream channel for a camera to send it's audio/video to.

I learned we can create multiple live stream channels.  We start with a Google account, then we go to our account's list of channels `https://www.youtube.com/account`.  From here we create a new channel and click buttons until we get to Live Streaming.  It takes 24 hours for the live stream to be activated.  I learned this by browsing [How to Manage Multiple YouTube Channels: Tips and Tools](https://blog.hootsuite.com/multiple-youtube-channels/).
## Get a Permanent URL
- _TODO - Permanent URL_
I am waiting for 24 hours to go by so that I can use the fountain cam channel. I hope to use the info in the article [How to get a Permanent Link for a YouTube Livestream](https://techswift.org/2020/09/30/how-to-get-a-permanent-link-for-a-youtube-livestream/)...
# Raspberry Pi Stuff
This section gives us a high level view of the software running on the Rasp Pi that gets us to an encoding and stream that is then sent to YouTube over wifi.
## Software Flow
Here's the flow we'll be implementing
```
Camera pointed at Fountain ==> raspivid captures raw h.264 video stream (pipes into...)==> ffmpeg creates "fake" audio + copies h.264 video stream then outputs rtmp stream with YouTube Live Key ==> YouTube Live shows stream
```
The `raspivid` software is installed with the Rasp Pi OS.  It  is able to easily give us an `h264` video stream.  We then pipe the `h264` video stream to the `ffmpeg` software.  `ffmpeg` takes in the video and multiplexes it with a "fake" audio signal.  The audio/video channel is sent out onto the Internet by `ffmpeg` using the `RTMP` protocol and the key provided by my YouTube Live channel.

- _TODO: Audio_


Here's the `raspivid` and `ffmpeg` commands I used:
```
raspivid -o - -ih -t 0 -b 1000000 | ffmpeg -i - -f s16le -i /dev/zero -c:v copy -c:a aac -g 50 -f flv -flvflags no_duration_filesize rtmp://a.rtmp.youtube.com/live2/[YOUTUBE LIVE KEY]
```
Even after searching and reading other attempts at YouTube Live Streaming from a Raspberry Pie, I spent __a significant amount of time__ figuring out what commands work best.  The options used with the commands are documented below.
# Hardware
- [Rasp Pi Zero W](https://www.adafruit.com/product/3400) 
- [Power cord for Rasp Pi](https://www.adafruit.com/product/1995)
- [SD/MicroSD Card](https://www.adafruit.com/product/2693)
- [Rasp Pi Camera V2](https://www.adafruit.com/product/3099)
- [Rasp Pi Zero Camera Cable](https://www.adafruit.com/product/3157)

_The step of getting the hardware connected/up and running is well documented in other places._
- TODO - Add mic and audio once video is understood and working.
# Software
## Install Rasp Pi and Camera Module
_Follow the many [excellent guides](https://www.raspberrypi.org/documentation/installation/installing-images/) for installing Rasp Pi.  I tend to install the lite version and ssh in.  It does seem having the desktop makes some stuff easier, so having a spare monitor/keyboard just for a pi if I was dong many projects would be a convenient way to go..The camera software/config needs to be turned on. There are many great guides for doing this_
- don't forget to put a file named `ssh` on the boot micro SD card.
- don't forget to put a `wpa_supplicant.conf` file on the boot micro SD car.

Here's the contents of the `wpa_supplicant.conf` file I use.  _Note: ssid, psk, id will need to be changed to the current environement._
```
country=US
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1

network={
    ssid="name of wifi"
    psk="password"
    id_str="home"
}
network={
    ssid="another wifi"
    psk="its password"
    id_str="an id string"
}
```
I then use `Angry IP` software to figure out the rasp pi's IP address.  Then I ssh in with something like
```
 ssh pi@192.168.86.36
```

### Update the OS

As noted in [the documentation](https://www.raspberrypi.org/documentation/raspbian/updating.md), _First, update your system's package list by entering the following command:_
```
sudo apt update
```
_Next, upgrade all your installed packages to their latest versions with the following command:_
```
sudo apt full-upgrade
```
_Generally speaking, doing this regularly will keep your installation up to date for the particular major Raspberry Pi OS release you are using..._
- TODO: Makes sense to periodically do `sudo apt full-upgrade` using a `cron` job.

## Install ffmpeg

### Preferred Way
```
sudo apt install ffmpeg
```
### The Hard Way
__Avoid this method.  I probably include it mostly because of the pain it caused.__
For my first attempt, I installed `ffmpeg` according to the directions provided by [Pi My Life](https://pimylifeup.com/compiling-ffmpeg-raspberry-pi/comment-page-1/?unapproved=56712&moderation-hash=7fef9638fd78bfff1786506e5bb24892#comment-56712).

This took over 9 hours and at the end I still had problems with `make` that I was able to resolve based on my previous experience with `make`.

I went this way first because I was overwhelmed and clueless.  I googled `fmpeg` and didn't realize I could just have done the first way.
## Install SMB (optional)
_Installing SMB services __might__ be useful, but not necessary_
It is pleasing to me to view Rasp Pi files as if the Rasp Pi files were on a networked hard drive.  To do this, I followed [these instructions](https://pimylifeup.com/raspberry-pi-samba/). 
# raspivid Options

As described in [the Raspberry Pi documentation for the camera module](https://www.raspberrypi.org/documentation/raspbian/applications/camera.md), raspivid is for capturing video.  What is super great about this software is it works seamlessly with the Rasp pi camera to encode the format of video - h264- that can be bit copied into the stream sent to YouTube live _in hardware_.  Isn't that awesome?  This means so much more is possible with such a small little machine like our Zero W!

The `raspivid` command:
```
raspivid -o - -ih -t 0 -b 1000000
```
from `raspivid --help` :
- -o : output filename.   '-' means output to stdout. _Makes sense since output will be piped to ffmpeg._
- -ih : Forces the stream to include PPS and SPS headers on every I-frame. Needed for certain streaming cases e.g. Apple HLS. These headers are small, so don't greatly increase the file size. __NOTE: Without this option, the `raspivid` h.264 "pipe" into `ffmpeg` broke after random hours, usually anywhere between 2 and 12 hours__.
- -t 0 : Time (in ms) to capture for. If not specified, set to 5s. Zero to disable.  _Again makes sense, since we want 7/24 streaming._
- -b :  Set bitrate. Use bits per second (e.g. 10MBits/s would be -b 10000000). The key here was a bitrate of 1M, which seems to be the lowest YouTube Live Stream will reasonably take without being super pissed.  Even at 1M, it wants me to bump up streaming to at least 4M...but since I'm not trying to influence anyone with a super quality stream, I'm sticking to 1M under the assumption the poor hamsters in the Rasp Pi won't have to work as hard.

- TODO: Not using but might need (most examples include) -vf and -hf : vertical and horizontal flip.  I guess the rasp pi camera needs this.  I don't know.
# ffmpeg Options
The `ffmpeg` command:
```
ffmpeg -i - -f s16le -i /dev/zero -c:v copy -c:a aac -g 50 -f flv -flvflags no_duration_filesize rtmp://a.rtmp.youtube.com/live2/[YOUTUBE LIVE KEY]
```

# cron
## Logging
To see if the `cron` job executed, the ` /etc/rsyslog.conf` needs to be modified as noted [on a Rasp Pi forum posting](https://www.raspberrypi.org/forums/viewtopic.php?t=186833):
```
 sudo nano /etc/rsyslog.conf
```
_and look for the section Rules and uncomment the line_
```
 # cron.*                          /var/log/cron.log
```
_save the file and reboot your pi and cron logging should be on and logging to : /var/log/cron.log_

## Running the Stream
We're going to use the `systemd` service to start and stop our RTMP stream.  We'll need three 'types' of files:
- a `systemd` service file that lets `systemd` know what we are running.
- a shell script that the `systemd` service file invokes.  
- one or more `systemd` timer files to let `systemd` know when and how to stop/start the service.
### What is systemd
To get us up to speed on systemd:
* [Intro to systemd video](https://youtu.be/AtEqbYTLHfs?t=147).  
  * [Place in the video where using commands starts](https://youtu.be/AtEqbYTLHfs?t=230).
* [Article on how to autorun service using systemd](https://www.raspberrypi-spy.co.uk/2015/10/how-to-autorun-a-python-script-on-boot-using-systemd/).
* [Nice article on useful `systemd` commands](https://www.digitalocean.com/community/tutorials/how-to-use-systemctl-to-manage-systemd-services-and-units)
___
### Before we Begin
___
_Before we begin with `systemd`, I wanted to point out it might be "poor practice" to "dump" files in the directory where `systemd` looks.  But since my Rasp Pi escapades are so...so...so DIY, I have a "what the heck" attitude.  You might want to put more thought into which directories your files are placed._

Let's walk through the steps

______
STauFF
***************************************
tsting writing to file...couldn't get it working.  first it was perms.  Had to give rw perms to both test.log and directory.  then it was something about the service file.  added [workingDirectory=~](https://askubuntu.com/questions/1063153/systemd-service-working-directory-not-change-the-directory) this solved it.

Nice guide for a few great journalctl commands (like -xe) https://linuxhandbook.com/journalctl-command/

systemd is getting too heavy for this...looking into [cron](https://en.wikipedia.org/wiki/Cron)

--- latest plan
use cron
- on reboot start video stream
- after two hours reboot.


# crontab

- checking if cron is running task, writing to stdout and stderr:
https://www.thegeekstuff.com/2012/07/crontab-log/

cat /var/log/cron.log

top command to a file:
https://www.tecmint.com/save-top-command-output-to-a-file/

 ____
### Go to the systemd directory
___
`cd /lib/systemd/system` This is where `systemd` looks for it's scripts.
___
### Create the `systemd` service file
___
`systemd` works by reading what it is to do from instructions within a `.service` file.  So let's get this done!
- `sudo nano birdcam.service` Use  `nano` editor to create a script whose filename is `birdcam.service`.
- copy/paste a template `systemd` service file that I have used in the past since I forget most of what I have learned.
```
[Unit]
Description=Birdcam
# Don't start the service until the network is up and running
After=network.target

[Service]
# See old project in github (https://github.com/BitKnitting/should_I_water)
# Thinking we don't need an environment file...
# EnvironmentFile=/home/pi/myLadybugHelper/env_variables/environment_siw
ExecStart=/lib/systemd/system/startbirdcam.sh
Restart=on-failure
User=pi

##[Install]
# Components of this application should be started at boot time
WantedBy=multi-user.target
```
Exit and save the file.  
___
- We don't use any environment variables like we might if we were running a Python script.
- ExecStart points to a shell script.  `systemd` does not implement a full shell.  For example, it does not support the pipe command.
- The [Install] section is commented out because starting the service (at boot) will be handled by a .timer file.
___
### Create the Shell Script File
___
 We'll create the birdcam.sh script file to contain our `birdcam stream` stuff.
- `sudo nano birdcam.sh` (Using the default `nano` editor).
- copy / paste:
```
#!/bin/bash
# Remember to replace YOUTUBE LIVE KEY!
exec raspivid -o - -t 0 -b 1000000 | ffmpeg -i - -f s16le -i /dev/zero -c:v copy -c:a aac -g 50 -f flv -flvflags no_duration_filesize rtmp://a.rtmp.youtube.com/live2/[YOUTUBE LIVE KEY]
```
__REMEMBER TO REPLACE [YOUTUBE LIVE KEY] with your YouTube Live Key!__
Exit and save the file
- `sudo chmod +x birdcam.sh` sets the permissions so that `systemd` can execute the shell script.
___
### Create the start service timer file
___
```
[Unit]
Description=start the birdcam service at boot and again at the time after boot noted in OnUnitActiveSec

[Timer]
Unit=startbirdcam.service
OnBootSec=1min
OnUnitActiveSec=3min

[Install]
WantedBy=timers.target
```
The timers file says start the birdcam service 1 minute after timers.target boots and then run it every 3 minutes after that. 
- enable with sudo systemctl enable the timer
#### Step 4: Start Streaming
##### Step 4A: Start YouTube Live Streaming
Make sure your YouTube Live Streaming page is reading and waiting for an `RTMP` URL coming from our Rasp Pi!
##### Step 4B: Start the birdcam Service
```
sudo systemctl start bircam.service
````
Check to see if it's running ok...
```
$ systemctl status birdcam.service
● birdcam.service - Birdcam
   Loaded: loaded (/lib/systemd/system/birdcam.service; disabled; vendor preset: enabled)
   Active: active (running) since Sat 2021-05-22 15:01:37 BST; 6min ago
 Main PID: 1859 (birdcam.sh)
    Tasks: 12 (limit: 725)
   CGroup: /system.slice/birdcam.service
           ├─1859 /bin/bash /lib/systemd/system/birdcam.sh
           ├─1860 raspivid -o - -t 0 -b 1000000
           └─1861 ffmpeg -i - -f s16le -i /dev/zero -c:v copy -c:a aac -g 50 -f flv -flvflags no_duration_filesize rtmp://a.rtmp.youtube.com/live2/[MY KEY WAS HERE]

May 22 15:01:42 raspcam birdcam.sh[1859]: Output #0, flv, to 'rtmp://a.rtmp.youtube.com/live2/[MY KEY WAS HERE]':
May 22 15:01:42 raspcam birdcam.sh[1859]:   Metadata:
May 22 15:01:42 raspcam birdcam.sh[1859]:     encoder         : Lavf59.2.100
May 22 15:01:42 raspcam birdcam.sh[1859]:   Stream #0:0: Video: h264 (High) ([7][0][0][0] / 0x0007), yuv420p(progressive), 1920x1080, q=2-31, 25 fps, 25 tbr, 1k tbn
May 22 15:01:42 raspcam birdcam.sh[1859]:   Stream #0:1: Audio: aac (LC) ([10][0][0][0] / 0x000A), 44100 Hz, mono, fltp, 69 kb/s
May 22 15:01:42 raspcam birdcam.sh[1859]:     Metadata:
May 22 15:01:42 raspcam birdcam.sh[1859]:       encoder         : Lavc59.1.100 aac
May 22 15:01:42 raspcam birdcam.sh[1859]: [202B blob data]
May 22 15:01:42 raspcam birdcam.sh[1859]: [flv @ 0x2ae5550] Timestamps are unset in a packet for stream 0. This is deprecated and will stop working in the future. Fix your code to set t
May 22 15:05:56 raspcam birdcam.sh[1859]: [48.0K blob data]
```
Use the Process ID (PID) of the birdcam.sh shell script to get journaling info...
```
$ journalctl _PID=1859
-- Logs begin at Sun 2021-05-23 04:17:01 BST, end at Sun 2021-05-23 13:17:13 BST. --
May 23 13:17:09 raspcam birdcam.sh[629]: ffmpeg version git-2021-05-16-f53414a Copyright (c) 2000-2021 the FFmpeg develo
May 23 13:17:09 raspcam birdcam.sh[629]:   built with gcc 8 (Raspbian 8.3.0-6+rpi1)
May 23 13:17:09 raspcam birdcam.sh[629]:   configuration: --extra-cflags=-I/usr/local/include --extra-ldflags=-L/usr/loc
May 23 13:17:09 raspcam birdcam.sh[629]:   libavutil      57.  0.100 / 57.  0.100
May 23 13:17:09 raspcam birdcam.sh[629]:   libavcodec     59.  1.100 / 59.  1.100
May 23 13:17:09 raspcam birdcam.sh[629]:   libavformat    59.  2.100 / 59.  2.100
May 23 13:17:09 raspcam birdcam.sh[629]:   libavdevice    59.  0.100 / 59.  0.100
May 23 13:17:09 raspcam birdcam.sh[629]:   libavfilter     8.  0.101 /  8.  0.101
May 23 13:17:09 raspcam birdcam.sh[629]:   libswscale      6.  0.100 /  6.  0.100
May 23 13:17:09 raspcam birdcam.sh[629]:   libswresample   4.  0.100 /  4.  0.100
May 23 13:17:09 raspcam birdcam.sh[629]:   libpostproc    56.  0.100 / 56.  0.100
May 23 13:17:13 raspcam birdcam.sh[629]: Input #0, h264, from 'pipe:':
May 23 13:17:13 raspcam birdcam.sh[629]:   Duration: N/A, bitrate: N/A
May 23 13:17:13 raspcam birdcam.sh[629]:   Stream #0:0: Video: h264 (High), yuv420p(progressive), 1920x1080, 25 fps, 25
May 23 13:17:13 raspcam birdcam.sh[629]: Guessed Channel Layout for Input Stream #1.0 : mono
May 23 13:17:13 raspcam birdcam.sh[629]: Input #1, s16le, from '/dev/zero':
May 23 13:17:13 raspcam birdcam.sh[629]:   Duration: N/A, bitrate: 705 kb/s
May 23 13:17:13 raspcam birdcam.sh[629]:   Stream #1:0: Audio: pcm_s16le, 44100 Hz, mono, s16, 705 kb/s
May 23 13:17:13 raspcam birdcam.sh[629]: Stream mapping:
May 23 13:17:13 raspcam birdcam.sh[629]:   Stream #0:0 -> #0:0 (copy)
May 23 13:17:13 raspcam birdcam.sh[629]:   Stream #1:0 -> #0:1 (pcm_s16le (native) -> aac (native))
May 23 13:17:13 raspcam birdcam.sh[629]: Output #0, flv, to 'rtmp://a.rtmp.youtube.com/live2/bucf-ah5k-p9rr-bvf8-1xjk':
May 23 13:17:13 raspcam birdcam.sh[629]:   Metadata:
May 23 13:17:13 raspcam birdcam.sh[629]:     encoder         : Lavf59.2.100
May 23 13:17:13 raspcam birdcam.sh[629]:   Stream #0:0: Video: h264 (High) ([7][0][0][0] / 0x0007), yuv420p(progressive)
May 23 13:17:13 raspcam birdcam.sh[629]:   Stream #0:1: Audio: aac (LC) ([10][0][0][0] / 0x000A), 44100 Hz, mono, fltp,
May 23 13:17:13 raspcam birdcam.sh[629]:     Metadata:
May 23 13:17:13 raspcam birdcam.sh[629]:       encoder         : Lavc59.1.100 aac
```


### Stuff that Needs To be Installed



# Software
Obviously, Rasp Pi needs to be installed.  


  These are covered in many articles, like [this one](https://www.digikey.com/en/maker/blogs/streaming-live-to-youtube-and-facebook-using-raspberry-pi-camera)

## SMB
Being able to view an image or video on my Windows PC was achieved by installing SMB stuff on the Rasp Pi.  I think I used [these instructions](https://pimylifeup.com/raspberry-pi-samba/).  Now I can open file manager and view the files on the Rasp Pi as a network volume - just like a NAS server.

## Streaming Software
Caveat with my description: This is my first time working with live streaming.  Whether it be on a Raspberry Pi, PC, Mac, etc.... hmmmm....
In order to stream video from the raspberry pi camera, two pieces of software are piped together:

- ffmpeg: Encodes the video into h.264 format and streams it to a rtmp URL.

### raspivid
After playing around with video for a YouTube stream, I came up with these set of options:
```
raspivid -o - -t 0 -b 1000000 -vf -hf 
```
from `raspivid --help` :
- -o : output filename.   '-' means output to stdout. _Makes sense since output will be piped to ffmpeg._
- -t 0 : Time (in ms) to capture for. If not specified, set to 5s. Zero to disable.  _Again makes sense, since we want 7/24 streaming._
- -vf and -hf : vertical and horizontal flip.  I guess the rasp pi camera needs this.  I don't know.
- -b :  Set bitrate. Use bits per second (e.g. 10MBits/s would be -b 10000000). The key here was a bitrate of 1M, which seems to be the lowest YouTube Live Stream will reasonably take without being super pissed.  Even at 1M, it wants me to bump up streaming to at least 4M...but since I'm not trying to influence anyone with a super quality stream, I'm sticking to 1M under the assumption the poor hamsters in the Rasp Pi won't have to work as hard.

My goal with the raspivid options was to use as little as possible, assuming the wonderful folks that make this package are better suited at setting defaults.
#### Test...
As I watched a [video on ffmpeg](https://youtu.be/BiMP_hN8f6s), The speaker noted ffprobe -i <video/audio file> would give detailed info on the characteristics of the media in the file.  Let's check that out for the raspivid capture:
```
raspivid -o myvideo.tst -t 3000 -b 1000000 -vf -hf 
```
This creates the file myvideo.tst which contains 3 seconds of video at 1Mbps.  What does ffprobe tell us?
```
ffprobe -i myvideo.tst -hide_banner

Input #0, h264, from 'myvideo.tst':
  Duration: N/A, bitrate: N/A
  Stream #0:0: Video: h264 (High), yuv420p(progressive), 1920x1080, 25 fps, 25 tbr, 1200k tbn
```
There is just one stream (File 0, Stream 0) - just video, no audio (no Stream 1 in the first file). The video (as expected since the Rasp Pi Camera hardware encodes video capture into h264 format) is in h264 format.  The resolution is 1920 x 1080 at 25 fps.  I don't know why the duration (3s) and bitrate (1M) were shown as N/A.  I'm not familiar with tbr and tbn, and I don't know if I care about yuv420p.  So moving right along....
### ffmpeg
...before we begin on using ffmpeg to create the RTMP stream that YouTube Live Server will pick up, there is an itsy bitsy problem.  YouTube Live Stream will not pick up our stream unless there is audio.  As we showed in the above ffprobe, we only have a video stream coming out of the camera.  So as in life, we have to fake it till we make it.  Or perhaps better said, make it a fake it...hmmm....

On to ffmpeg...What a beast, am I right?  So powerful...yet, wow - talk about a ton of options!  Thanks to raspivid, we have a video stream in raw h264 format that we'll pipe to ffmpeg.  We want ffmpeg to:
- copy the h264 instead of doing some type of transcoding (no processing)
- create an audio channel
 figure out how to fake an audio channel, combine the two streams and send them out on the Internet via a RTMP url that YouTube Live Stream picks up.

ffmpeg -i - -c:v copy 

video stream: 
- -i - : Take the input from stdin.
- -c:v copy : Copy the video stream.

audio stream:
- -i /dev/zero
- -c is for the codec.  c:v - a video codec.  c:a an audio codec

RTMP:
- f flv <rtmp stream>

[video showing  gop or group of pictures](https://youtu.be/BiMP_hN8f6s?t=531)
[video explaining gop](https://youtu.be/BiMP_hN8f6s?t=1066)

raspivid -o - -t 0 -b 1000000 -vf -hf | ffmpeg   -ar 44100  -c:a pcm_s16le -f s16le -ac 2 -i /dev/zero  -c:a aac -b:a 128k -f h264 -i - -c:v copy  -g 60 -strict experimental -f flv rtmp://a.rtmp.youtube.com/live2/bucf-ah5k-p9rr-bvf8-1xj

raspivid -o - -t 0 -b 1000000 -vf -hf | ffmpeg   -ar 44100  -c:a aac   -i /dev/zero   -f h264 -i - -c:v copy  -g 60 -strict experimental -f flv rtmp://a.rtmp.youtube.com/live2/bucf-ah5k-p9rr-bvf8-1xj


ffmpeg    -c:a pcm_s16le -f s16le  -i /dev/zero  -c:a aac test.aac

raspivid -o - -t 0 -b 1000000 -fps 30 -vf -hf | ffmpeg   -c:a pcm_s16le -f s16le  -i /dev/zero -c:a aac  -f h264 -i - -c:v copy  -g 60 -strict experimental -f flv rtmp://a.rtmp.youtube.com/live2/bucf-ah5k-p9rr-bvf8-1xj


ffmpeg audio raw types: https://trac.ffmpeg.org/wiki/audio%20types


========
Got this in YouTube:
```
Suggestion
The audio stream's current bitrate (0) is lower than the recommended bitrate. We recommend that you use an audio stream bitrate of 128 Kbps.
```
so made the audio stream mono instead of stereo by removing -ac 2
```
raspivid -o - -t 0 -b 1000000 -vf -hf | ffmpeg   -ar 44100  -c:a pcm_s16le -f s16le  -i /dev/zero  -c:a aac -ab 128k -f h264 -i - -c:v copy  -g 50 -strict experimental -f flv rtmp://a.rtmp.youtube.com/live2/bucf-ah5k-p9rr-bvf8-1xj


raspivid -o - -t 0 -b 1000000 | ffmpeg   -ar 44100  -acodec pcm_s16le -f s16le -ac 2 -i /dev/zero  -f h264 -i - -vcodec copy -acodec aac -ab 128k -g 20 -strict experimental -f flv rtmp://a.rtmp.youtube.com/live2/bucf-ah5k-p9rr-bvf8-1xjk

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
  475 pi        20   0  126644  25404  20328 R  41.5   6.8   0:21.66 ffmpeg
   56 root      20   0       0      0      0 I  21.2   0.0   0:06.24 kworker/u2:1-brcmf_wq/mmc1:0001:1
   44 root      20   0       0      0      0 I   4.6   0.0   0:01.77 kworker/0:2-events
  474 pi        20   0   63116   2456   1852 S   1.6   0.7   0:00.78 raspivid


  frame= 2983 fps= 31 q=-1.0 size=   12308kB time=00:01:59.28 bitrate= 845.3kbits/s speed=1.26x

  ===
MONO

raspivid -o - -t 0 -b 1000000 | ffmpeg   -ar 44100  -acodec pcm_s16le -f s16le  -i /dev/zero  -f h264 -i - -vcodec copy -acodec aac -ab 128k -g 20 -strict experimental -f flv rtmp://a.rtmp.youtube.com/live2/bucf-ah5k-p9rr-bvf8-1xjk

  483 pi        20   0  126544  25160  20208 S  31.7   6.7   0:24.06 ffmpeg
   56 root      20   0       0      0      0 I  23.0   0.0   0:10.43 kworker/u2:1-brcmf_wq/mmc1:0001:1
   49 root      20   0       0      0      0 I   4.9   0.0   0:01.89 kworker/0:3-events
  482 pi        20   0   63116   2440   1836 S   1.6   0.6   0:01.16 raspivid

frame= 2922 fps= 32 q=-1.0 size=   12055kB time=00:01:56.86 bitrate= 845.0kbits/s speed=1.26x

===
TAKE OUT 128k, put g to 50

raspivid -o - -t 0 -b 1000000 | ffmpeg   -ar 44100  -acodec pcm_s16le -f s16le  -i /dev/zero  -f h264 -i - -vcodec copy -acodec aac  -g 50 -strict experimental -f flv rtmp://a.rtmp.youtube.com/live2/bucf-ah5k-p9rr-bvf8-1xjk


  PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
  478 pi        20   0  126536  25308  20356 S  30.1   6.7   0:20.23 ffmpeg
   56 root      20   0       0      0      0 I  13.8   0.0   0:07.21 kworker/u2:1-events_unbound
  109 root      20   0       0      0      0 I   8.0   0.0   0:05.74 kworker/u2:2-brcmf_wq/mmc1:0001:1
   49 root      20   0       0      0      0 I   4.5   0.0   0:00.66 kworker/0:3-events
  477 pi        20   0   63116   2460   1856 S   1.6   0.7   0:00.96 raspivid
  510 pi        20   0   10172   2876   2420 R   1.3   0.8   0:00.15 top

frame= 2631 fps= 32 q=-1.0 size=   10851kB time=00:01:45.23 bitrate= 844.7kbits/s speed=1.27x

===
USE c: instead of acodec and vcodec

raspivid -o - -t 0 -b 1000000 | ffmpeg   -ar 44100  -c:a pcm_s16le -f s16le  -i /dev/zero  -f h264 -i - -c:v copy -c:a aac  -g 50 -strict experimental -f flv rtmp://a.rtmp.youtube.com/live2/bucf-ah5k-p9rr-bvf8-1xjk

===
TAKE out -ar 44100 

WINNER WINNER CHICKEN DINNER....



raspivid -o - -t 0 -b 1000000 | ffmpeg    -c:a pcm_s16le -f s16le  -i /dev/zero  -f h264 -i - -c:v copy -c:a aac  -g 50 -strict experimental -f flv rtmp://a.rtmp.youtube.com/live2/bucf-ah5k-p9rr-bvf8-1xjk


  PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
  483 pi        20   0  126544  25260  20320 S  30.0   6.7   0:24.15 ffmpeg
   56 root      20   0       0      0      0 I  19.7   0.0   0:09.29 kworker/u2:1-brcmf_wq/mmc1:0001:1
   49 root      20   0       0      0      0 I   4.2   0.0   0:03.16 kworker/0:3-events
  109 root      20   0       0      0      0 I   2.3   0.0   0:06.43 kworker/u2:2-brcmf_wq/mmc1:0001:1
  482 pi        20   0   63116   2440   1836 S   1.3   0.6   0:01.23 raspivid

  frame= 2917 fps= 32 q=-1.0 size=   12033kB time=00:01:56.68 bitrate= 844.8kbits/s s

frame= 6740 fps= 31 q=-1.0 size=   27797kB time=00:04:29.58 bitrate= 844.7kbits/s speed=1.23x

  Input #0, s16le, from '/dev/zero':
  Duration: N/A, bitrate: 705 kb/s
  Stream #0:0: Audio: pcm_s16le, 44100 Hz, mono, s16, 705 kb/s
Input #1, h264, from 'pipe:':
  Duration: N/A, bitrate: N/A
  Stream #1:0: Video: h264 (High), yuv420p(progressive), 1920x1080, 25 fps, 25 tbr, 1200k tbn
Stream mapping:
  Stream #1:0 -> #0:0 (copy)
  Stream #0:0 -> #0:1 (pcm_s16le (native) -> aac (native))
Output #0, flv, to 'rtmp://a.rtmp.youtube.com/live2/bucf-ah5k-p9rr-bvf8-1xjk':
  Metadata:
    encoder         : Lavf59.2.100
  Stream #0:0: Video: h264 (High) ([7][0][0][0] / 0x0007), yuv420p(progressive), 1920x1080, q=2-31, 25 fps, 25 tbr, 1k tbn
  Stream #0:1: Audio: aac (LC) ([10][0][0][0] / 0x000A), 44100 Hz, mono, fltp, 69 kb/s
    Metadata:
      encoder         : Lavc59.1.100 aac
[h264 @ 0x3679790] Thread message queue blocking; consider raising the thread_queue_size option (current value: 8)
[flv @ 0x3790c30] Timestamps are unset in a packet for stream 0. This is deprecated and will stop working in the future. Fix your code to set the timestamps properly

at about 2 minutes in , YouTube live stream complains:

The stream's current bitrate (843.20 Kbps) is lower than the recommended bitrate. We recommend that you use a stream bitrate of 4500 Kbps.
----
about 2 1/2 to 31/2 hours in:
av_interleaved_write_frame(): End of filekB time=02:49:40.76 bitrate= 844.6kbits/s speed= 1.2x
[flv @ 0x3790c30] Failed to update header with correct duration.
[flv @ 0x3790c30] Failed to update header with correct filesize.
Error writing trailer of rtmp://a.rtmp.youtube.com/live2/bucf-ah5k-p9rr-bvf8-1xjk: End of file
frame=254521 fps= 30 q=-1.0 Lsize= 1049688kB time=02:49:40.83 bitrate= 844.6kbits/s speed= 1.2x
video:1035700kB audio:1738kB subtitle:0kB other streams:0kB global headers:0kB muxing overhead: 1.180841%
[aac @ 0x36b1680] Qavg: 65535.953
Conversion failed!
```
_[Note from stackoverflow](https://stackoverflow.com/questions/50300013/ffmpeg-streaming-stops-after-few-seconds)_

one person's ffmpeg having  this problem

BEFORE

-f dshow -i audio="Microphone device" -strict -2 -c:a aac -f gdigrab -framerate 30 -offset_x 161 -offset_y 203 -video_size 430x322 -show_region 1 -i title="Window Title" -b:v 415K -g 60 -keyint_min 60 -b:a 32K -ar 22050 -filter:a "volume=0.8" -pix_fmt yuv420p -c:v libx264 -preset medium -bufsize 400k -maxrate 400k -f flv rtmp://rtmpurl/stream

AFTER
 I don't know why, but if I put the video input in front of the audio input, works fine. This is my fixed command, it's still not perfect, but it works:

-f gdigrab -framerate 30 -offset_x 161 -offset_y 203 -video_size 430x322 -show_region 1 -i title="Window Title" -f dshow -i audio="Microphone Device" -strict -2 -c:a aac -b:v 415K -g 60 -keyint_min 60 -b:a 32K -ar 22050 -filter:a "volume=0.6499733" -pix_fmt yuv420p -c:v libx264 -preset medium -bufsize 4000k -maxrate 4000k -f flv -flvflags no_duration_filesize rtmp://rtmpurl/stream

MINE
 ffmpeg    -c:a pcm_s16le -f s16le  -i /dev/zero  -f h264 -i - -c:v copy -c:a aac  -g 50 -strict experimental -f flv rtmp://a.rtmp.youtube.com/live2/bucf-ah5k-p9rr-bvf8-1xjk

 try
 ffmpeg  -f h264 -i - -c:v copy -c:a aac -g 50 -f s16le c:a pcm_s16le -i /dev/zero -f flv rtmp://a.rtmp.youtube.com/live2/bucf-ah5k-p9rr-bvf8-1xjk

 so combining:
 raspivid -o - -t 0 -b 1000000 | ffmpeg  -f h264 -i - -c:v copy -c:a aac -g 50 -f s16le c:a pcm_s16le -i /dev/zero -f flv rtmp://a.rtmp.youtube.com/live2/bucf-ah5k-p9rr-bvf8-1xjk

 --- USING THIS MAY 21... ----
 Start at 7:20 AM....

 raspivid -o - -t 0 -b 1000000 | ffmpeg -i - -f s16le -i /dev/zero -c:v copy -c:a aac -g 50 -f flv rtmp://a.rtmp.youtube.com/live2/bucf-ah5k-p9rr-bvf8-1xjk
 ++++
 ADD the -flvflags no_duration_filesize 
 ----
  raspivid -o - -t 0 -b 1000000 | ffmpeg -i - -f s16le -i /dev/zero -c:v copy -c:a aac -g 50 -f flv -flvflags no_duration_filesize rtmp://a.rtmp.youtube.com/live2/bucf-ah5k-p9rr-bvf8-1xjk

 2  25896  21104 S  31.1   6.9   0:31.19 ffmpeg
   56 root      20   0       0      0      0 I  22.1   0.0   0:12.52 kworker/u2:1-brcmf_wq/mmc1:0001:1
   44 root      20   0       0      0      0 I   4.5   0.0   0:02.37 kworker/0:2-events
  109 root      20   0       0      0      0 I   2.2   0.0   0:08.87 kworker/u2:2-events_unbound
  482 pi        20   0   63116   2528   1924 S   1.9   0.7   0:01.51 raspivid

  Input #0, h264, from 'pipe:':
  Duration: N/A, bitrate: N/A
  Stream #0:0: Video: h264 (High), yuv420p(progressive), 1920x1080, 25 fps, 25 tbr, 1200k tbn
Guessed Channel Layout for Input Stream #1.0 : mono
Input #1, s16le, from '/dev/zero':
  Duration: N/A, bitrate: 705 kb/s
  Stream #1:0: Audio: pcm_s16le, 44100 Hz, mono, s16, 705 kb/s
Stream mapping:
  Stream #0:0 -> #0:0 (copy)
  Stream #1:0 -> #0:1 (pcm_s16le (native) -> aac (native))
Output #0, flv, to 'rtmp://a.rtmp.youtube.com/live2/bucf-ah5k-p9rr-bvf8-1xjk':
  Metadata:
    encoder         : Lavf59.2.100
  Stream #0:0: Video: h264 (High) ([7][0][0][0] / 0x0007), yuv420p(progressive), 1920x1080, q=2-31, 25 fps, 25 tbr, 1k tbn
  Stream #0:1: Audio: aac (LC) ([10][0][0][0] / 0x000A), 44100 Hz, mono, fltp, 69 kb/s
    Metadata:
      encoder         : Lavc59.1.100 aac
[h264 @ 0x30811a0] Thread message queue blocking; consider raising the thread_queue_size option (current value: 8)
[flv @ 0x30911e0] Timestamps are unset in a packet for stream 0. This is deprecated and will stop working in the future. Fix your code to set the timestamps properly
frame= 4438 fps= 31 q=-1.0 size=   18304kB time=00:02:57.51 bitrate= 844.7kbits/s speed=1.24x




two input streams:
- s161e for audio
- h264 for video
These are muxed for the flv file format
flv             FLV (Flash Video)

[about RTMP](https://www.wowza.com/blog/rtmp-streaming-real-time-messaging-protocol)
The RTMP specification is a streaming protocol initially designed for the transmission of audio, video, and other data between a dedicated streaming server and the Adobe Flash Player. 

this works:
ffmpeg   -f s16le  -i /dev/zero test.aac
try:
raspivid -o - -t 0 -b 1000000 | ffmpeg  -f h264 -i - test.mp4

i.e.: don't need initial codec...

-f = file format with input at -i
[from doc](https://ffmpeg.org/ffmpeg.html#toc-Main-options):

ffmpeg is a very fast video and audio converter 

So we input and then we output into some format....The codec (c:) says what the encoded output will be.

The format option may be needed for raw input files.

-f fmt (input/output)
Force input or output file format. The format is normally auto detected for input files and guessed from the file extension for output files, so this option is not needed in most cases.

(at a command line, type in ffmpeg -formats and a slew of different audio/video formats will show up)
```
h264            raw H.264 video
s16le           PCM signed 16-bit little-endian

-c[:stream_specifier] codec (input/output,per-stream)
-codec[:stream_specifier] codec (input/output,per-stream)
Select an encoder (when used before an output file) or a decoder (when used before an input file) for one or more streams. codec is the name of a decoder/encoder or a special value copy (output only) to indicate that the stream is not to be re-encoded.
```

 First - capturing audio from a s161e device.  The audio device is /dev/zero
 Second - capturing video from a h264 device.  The video device is stdin.  There is no transcoding, just bitcopy in the video since it is already h264 encoded.

```
At the end of muxing a flv file, FFmpeg updates the header (at front of file) with duration and filesize values. However, when you are streaming, ffmpeg can't seek to the front, so the warnings are displayed.  You can disable this function, by adding a flag (-flvflags no_duration_filesize)...
```

--- try adding this:

raspivid -o - -t 0 -b 1000000 | ffmpeg    -c:a pcm_s16le -f s16le  -i /dev/zero  -f h264 -i - -c:v copy -c:a aac -g 50 -strict experimental -f flv -flvflags no_duration_filesize rtmp://a.rtmp.youtube.com/live2/bucf-ah5k-p9rr-bvf8-1xjk



  PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
  519 pi        20   0  126656  25924  20848 R  31.5   6.9   0:23.93 ffmpeg
   56 root      20   0       0      0      0 I  20.1   0.0   0:08.14 kworker/u2:1-brcmf_wq/mmc1:0001:1
  506 root      20   0       0      0      0 I   4.5   0.0   0:02.34 kworker/0:0-events
  109 root      20   0       0      0      0 I   2.9   0.0   0:02.74 kworker/u2:2-brcmf_wq/mmc1:0001:1
  518 pi        20   0   63116   2436   1832 S   1.6   0.6   0:01.16 raspivid

  -----

  NOTE: I DON't THINK THIS ADDRESSES THE REAL ISSUE WHICH IS:
  av_interleaved_write_frame(): End of file

  why did we get to an end of file on a stream?

frame=  734 fps= 37 q=-1.0 size=    3047kB time=00:00:29.35 bitrate= 850.4kbits/s speed=1.48x
[1]+  Stopped                 raspivid -o - -t 0 -b 1000000 | ffmpeg -ar 44100 -acodec pcm_s16le -f s16le -ac 2 -i /dev/zero -f h264 -i - -vcodec copy -acodec aac -ab 128k -g 20 -strict experimental -f flv rtmp://a.rtmp.youtube.com/live2/bucf-ah5k-p9rr-bvf8-1xjk





```
```
  483 pi        20   0  126540  25336  20388 S  27.3   6.7   0:36.51 ffmpeg
  109 root      20   0       0      0      0 I  20.6   0.0   0:14.72 kworker/u2:2-brcmf_wq/mmc1:0001:1
   44 root      20   0       0      0      0 I   4.2   0.0   0:03.41 kworker/0:2-events
  482 pi        20   0   63116   2464   1860 S   1.6   0.7   0:01.53 raspivid
```


Here are a couple of raspivid commands I found from googling:
[Digikey's article](https://www.digikey.com/en/maker/blogs/streaming-live-to-youtube-and-facebook-using-raspberry-pi-camera)
```
raspivid -o - -t 0 -vf -hf -fps 10 -b 500000
```
Here's another one I found:
```
raspivid -o - -t 0 -w 1920 -h 1080 -fps 25 -b 3000000 -rot 270 -g 50 -f
```
Looking at each parameter being passed in:
- -o for output filename.  Both use '-' for the filename, so output is sent to stdout. Makes sense since output will be piped to ffmpeg.
- -t 0 for running indefinitely.  Again makes sense, since the streaming on a birdcam should be all the time.
- -vf and -hf for vertical and horizontal flip.  I guess the rasp pi camera needs this.  I don't know.
- -w for set image width (values between 64 and 1920)
- -h for set image height (values between 64 and 1080)
- -fps for Frames Per Second to record.  (minimum is 2fps maximum is 30fps)
- -b for bitrates per second.
- -rot 270  .. I have no idea
- -g 50 ..This option is vaguely explained, but I don't understand it...
- -f ... hmmm...

Here's what makes sense to me so far:
```
raspivid -o - -t 0 -vf -hf -fps 10 -b 400000 
```
The intent:
- I eliminated those options that I truly didn't understand, like rot, g, and f.
- The fps is low - the assumption is it saves processing power/bandwidth?
- The bits per second was the lowest recommended according to the comment in the article [How to Live Stream to YouTube With a Raspberry Pi](https://www.makeuseof.com/tag/live-stream-youtube-raspberry-pi/#:~:text=%3A%20These%20can%20be%20used%20to,high%20definition%20resolution%20(1080p).&text=%3A%20Output%20bitrate%20limit.,YouTube's%20recommendation%20is%20400%2D600kbps.) _YouTube's recommendation is 400-600kbps. A lower figure will reduce upload bandwidth, in exchange for a lower quality video._
### ffmpeg
Examples I have looked at include:
#### Example 1
From [USB webcam live streaming using FFmpeg](https://www.raspberrypi.org/forums/viewtopic.php?t=212417)
```
ffmpeg -ar 44100 -ac 2 -acodec pcm_s16le -f s16le -ac 2 -i /dev/zero -f v4l2 -s 1920x1080 -r 30 -i /dev/video0 -codec:v h264 -g 60 -b:v 2500k -codec:a aac -ac 2 -ar 44100 -ab 128k -f flv rtmp://a.rtmp.youtube.com/live2/*your_streaming_key*
```

=== 
just putting notes here...

on raspivid let's try a lower width height of video instead of vf and vh...i.e.:
```
raspivid -o - -t 0 -w 64 -h 64  -fps 10 -b 400000
```

The explanation:
1. -ar 44100 -ac 2 -acodec pcm_s16le -f s16le -ac 2 -i /dev/zero : a dummy audio device that reports a valid audio format and sample rate for YouTube
2. -f v4l2 : video format output from your camera
3. -s 1920x1080 : camera's video resolution
4. -r 30 : video framerate (I changed it from 25 to 30)
5. -i /dev/video0 : input device (your webcam)
6. -codec:v h264 : use h264 video codec
7. -g 60 : group of pictures (GOP) interval (ideally twice your framerate, so 30x2)
8. -b:v 2500k : video bitrate, in this case is 2.5Mbps, aka 2,500k, aka 2500000
9. -codec:a aac -ac 2 -ar 44100 -ab 128k : audio stream options, but this one is not "real" as you're using an empty audio stream
10. -f flv : output format (Flash video)
11. rtmp://... : the RTMP stream you're using as your output

The reason for the fake audio is explained in the post: _if you're trying to live stream to YouTube you'll need to keep using that "dummy" audio option otherwise the stream will be rejected._
Using the above and adjusting for the frame rate I am using (10 fps) - _note: not sure about the video bitrate needing to be so high...but don't know for sure, so leaving at 2.5Mbps_:
```
ffmpeg -ar 44100 -ac 2 -acodec pcm_s16le -f s16le -ac 2 -i /dev/zero -f v4l2 -s 1920x1080 -r 10 -i /dev/video0 -codec:v h264 -g 20 -b:v 2500k -codec:a aac -ac 2 -ar 44100 -ab 128k -f flv rtmp://a.rtmp.youtube.com/live2/*your_streaming_key*
```
Combining, I get: 
```
raspivid -o - -t 0 -vf -hf -fps 10 -b 400000 |  ffmpeg -thread_queue_size 10240 -r  -re -ar 44100 -ac 2 -acodec pcm_s16le -f s16le -ac 2 -i /dev/zero -f h264 -i - -vcodec copy -acodec aac -ab 128k -g 20 -strict experimental -f flv rtmp://a.rtmp.youtube.com/live2/STREAM-KEY
```
===
new try: see w and h on raspivid as well as s on ffmpeg...trying to minimize video amount.
```
raspivid -o - -t 0 -w 64 -h 64  -fps 10 -b 400000 | ffmpeg  -r 10  -re -ar 44100 -s 640x360 -acodec pcm_s16le -f s16le -ac 2 -i /dev/zero -f h264 -i - -vcodec copy -acodec aac -ab 128k -g 20 -strict experimental -f flv rtmp://a.rtmp.youtube.com/live2/STREAM-KEY
```
Option video_size not found.

so take out s?
```
raspivid -o - -t 0 -b 1000000 | ffmpeg   -ar 44100  -acodec pcm_s16le -f s16le -ac 2 -i /dev/zero  -f h264 -i - -vcodec copy -acodec aac -ab 128k -g 20 -strict experimental -f flv rtmp://a.rtmp.youtube.com/live2/bucf-ah5k-p9rr-bvf8-1xjk
===
he stream's current bitrate (327.58 Kbps) is lower than the recommended bitrate. We recommend that you use a stream bitrate of 1000 Kbps.

The stream's current bitrate (1006.88 Kbps) is lower than the recommended bitrate. We recommend that you use a stream bitrate of 4500 Kbps.

stream health is good.. so try upping bitrate and removing more...

increasing bitrate works but then crashing happens...

frame= 1622 fps= 33 q=-1.0 size=   13358kB time=00:01:04.84 bitrate=1687.6kbits/s speed=1.31x

frame= 7130 fps= 31 q=-1.0 size=   58437kB time=00:04:45.16 bitrate=1678.7kbits/s speed=1.22x

bumped bitrate from 2M to 1M
frame= 1098 fps= 34 q=-1.0 size=    4543kB time=00:00:43.88 bitrate= 848.1kbits/s speed=1.38x
frame= 2465 fps= 32 q=-1.0 size=   10239kB time=00:01:38.59 bitrate= 850.7kbits/s speed=1.27x
frame= 7158 fps= 31 q=-1.0 size=   29556kB time=00:04:46.32 bitrate= 845.6kbits/s speed=1.22x
frame=15305 fps= 30 q=-1.0 size=   63205kB time=00:10:12.19 bitrate= 845.8kbits/s speed=1.21x
frame=19543 fps= 30 q=-1.0 size=   80657kB time=00:13:01.72 bitrate= 845.2kbits/s speed=1.21x
frame=34093 fps= 30 q=-1.0 size=  140732kB time=00:22:43.68 bitrate= 845.4kbits/s speed=1.21x
frame=81711 fps= 30 q=-1.0 size=  337246kB time=00:54:28.44 bitrate= 845.3kbits/s speed= 1.2x
frame=132298 fps= 30 q=-1.0 size=  546033kB time=01:28:11.92 bitrate= 845.3kbits/s speed= 1.2x
frame=220077 fps= 30 q=-1.0 size=  908320kB time=02:26:43.07 bitrate= 845.3kbits/s speed= 1.2x

Guessed Channel Layout for Input Stream #0.0 : stereo
Input #0, s16le, from '/dev/zero':
  Duration: N/A, bitrate: 1411 kb/s
  Stream #0:0: Audio: pcm_s16le, 44100 Hz, stereo, s16, 1411 kb/s
Input #1, h264, from 'pipe:':
  Duration: N/A, bitrate: N/A
  Stream #1:0: Video: h264 (High), yuv420p(progressive), 1920x1080, 25 fps, 25 tbr, 1200k tbn
Stream mapping:
  Stream #1:0 -> #0:0 (copy)
  Stream #0:0 -> #0:1 (pcm_s16le (native) -> aac (native))
Output #0, flv, to 'rtmp://a.rtmp.youtube.com/live2/bucf-ah5k-p9rr-bvf8-1xjk':
  Metadata:
    encoder         : Lavf59.2.100
  Stream #0:0: Video: h264 (High) ([7][0][0][0] / 0x0007), yuv420p(progressive), 1920x1080, q=2-31, 25 fps, 25 tbr, 1k tbn
  Stream #0:1: Audio: aac (LC) ([10][0][0][0] / 0x000A), 44100 Hz, stereo, fltp, 128 kb/s
    Metadata:
      encoder         : Lavc59.1.100 aac
[h264 @ 0x2e171a0] Thread message queue blocking; consider raising the thread_queue_size option (current value: 8)
[flv @ 0x2ef29f0] Timestamps are unset in a packet for stream 0. This is deprecated and will stop working in the future. Fix your code to set the timestamps properly



asionally at this bitrate YouTube doesn't get data...so it asks you to bump up the bitrate.  Becausee there was a brief period of "no data" (buffering)
YouTube errors:
stream health goes from poor to no data.
- WARNING: The stream's current bitrate (11.22 Kbps) is lower than the recommended bitrate. We recommend that you use a stream bitrate of 400 Kbps.
- WARNING: The current resolution (64 x 64) is not optimal
- Error: YouTube is not receiving enough video to maintain smooth streaming.
- WARNING: The bitrate (12.14 Kbps) is lower than recommended bitrate.
Is -ab 128k saying set audio bit rate to 128k?

so bitrate..guy says for 1080p, between 4.5 to 6Mbps... so lets bump up the bit rate from 12.14 to 100 Kbps, etc.
```
raspivid -o - -t 0 -w 64 -h 64  -fps 10 -b 400000 | ffmpeg  -r 10  -re -ar 44100  -acodec pcm_s16le -f s16dle -ac 2 -i /dev/zero -b:v 1M -f h264 -i - -vcodec copy -acodec aac -ab 128k -g 20 -strict experimental -f flv rtmp://a.rtmp.youtube.com/live2/bucf-ah5k-p9rr-bvf8-1xjk
```


ffmpeg info https://www.raspberrypi.org/forums/viewtopic.php?t=212417




A typical command I've seen is something like:
```
My first attempt worked.  But there was too much buffering.  The YouTube UI says:
```
Error
YouTube is not receiving enough video to maintain smooth streaming. As such, viewers will experience buffering.

```
My setup was based on [Digikey's article](https://www.digikey.com/en/maker/blogs/streaming-live-to-youtube-and-facebook-using-raspberry-pi-camera).  

The gobley gook command the article recommends:
```
 | ffmpeg -re -ar 44100 -ac 2 -acodec pcm_s16le -f s16le -ac 2 -i /dev/zero -f h264 -i - -vcodec copy -acodec aac -ab 128k -g 50 -strict experimental -f flv rtmp://a.rtmp.youtube.com/live2/STREAM-KEY
```
The Rasp Pi was gave a message:
```
[h264 @ 0x348a1e0] Thread message queue blocking; consider raising the thread_queue_size option (current value: 8)
[flv @ 0x34a56e0] Timestamps are unset in a packet for stream 0. This is deprecated and will stop working in the future. Fix your code to set the timestamps properly


frame= 8862 fps= 10 q=-1.0 size=   54608kB time=00:05:54.47 bitrate=1262.0kbits/s speed=0.406x
```
Then eventually failed:
```
[flv @ 0x34a56e0] Failed to update header with correct duration.
[flv @ 0x34a56e0] Failed to update header with correct filesize.
Error writing trailer of rtmp://a.rtmp.youtube.com/live2/bucf-ah5k-p9rr-bvf8-1xjk: End of file
frame=43501 fps= 10 q=-1.0 Lsize=  268102kB time=00:29:00.03 bitrate=1262.2kbits/s speed=0.401x
video:265565kB audio:443kB subtitle:0kB other streams:0kB global headers:0kB muxing overhead: 0.787225%
[aac @ 0x34c2fd0] Qavg: 65536.000
Conversion failed!
```

![figuring out video](figuring_out_video.png)

Looking at the video, I see the vertical and horizontal ARE flipped, so change (raspivid) command:
```
raspivid -o - -t 0 -b 1000000 -vf -hf | ffmpeg   -ar 44100  -acodec pcm_s16le -f s16le -ac 2 -i /dev/zero  -f h264 -i - -vcodec copy -acodec aac -ab 128k -g 20 -strict experimental -f flv rtmp://a.rtmp.youtube.com/live2/bucf-ah5k-p9rr-bvf8-1xjk

**** try

raspivid -o - -t 0 -b 1000000 -vf -hf | ffmpeg -re  -ar 44100  -acodec pcm_s16le -f s16le -ac 2 -i /dev/zero  -f h264 -i - -c:v copy -c:a aac -ab 128k -g 50 -strict experimental -f flv rtmp://a.rtmp.youtube.com/live2/bucf-ah5k-p9rr-bvf8-1xjk

raspivid -o - -t 0 -b 1000000 -vf -hf | ffmpeg   -ar 44100  -acodec pcm_s16le -f s16le -ac 2 -i /dev/zero  -f h264 -i - -vcodec copy -acodec aac -ab 128k -g 60 -strict experimental -f flv rtmp://a.rtmp.youtube.com/live2/bucf-ah5k-p9rr-bvf8-1xjk

[h264 @ 0x22851f0] Thread message queue blocking; consider raising the thread_queue_size option (current value: 8)
[flv @ 0x23a0600] Timestamps are unset in a packet for stream 0. This is deprecated and will stop working in the future. Fix your code to set the timestamps properly
av_interleaved_write_frame(): End of filekB time=03:21:09.22 bitrate= 845.3kbits/s speed=   1x
[flv @ 0x23a0600] Failed to update header with correct duration.
[flv @ 0x23a0600] Failed to update header with correct filesize.
Error writing trailer of rtmp://a.rtmp.youtube.com/live2/bucf-ah5k-p9rr-bvf8-1xjk: End of file
frame=301741 fps= 25 q=-1.0 Lsize= 1245403kB time=03:21:09.63 bitrate= 845.3kbits/s speed=   1x
video:1227804kB audio:3075kB subtitle:0kB other streams:0kB global headers:0kB muxing overhead: 1.179903%
[aac @ 0x22c07e0] Qavg: 65536.000
```
from https://www.reddit.com/r/raspberry_pi/comments/hr3hzz/youtube_live_streaming_aquarium_underwater_camera/
_you can just do -c:v copy and it won't waste any CPU power trying to encode anything. It runs best if you tell ffmpeg what framerate raspivid is producing frames at, and the -re option is important to keep it going at the right speed._

## ffmpeg
### One Way
I ended up compiling everything from github following the steps [in this article](https://pimylifeup.com/compiling-ffmpeg-raspberry-pi/)...that took A LONG LONG TIME....I'm thinking around 9-12 hours.
### Another Way
Then I read the article [Streaming Live to YouTube and Facebook using Raspberry Pi Camera](https://www.digikey.com/en/maker/blogs/streaming-live-to-youtube-and-facebook-using-raspberry-pi-camera)..where in a few steps, there was an FFMPEG...sigh...but then again...not sure it would work.  The article is from 2017 and it is 2021.
  506 pi        20   0  126628  25424  20376 R  42.9   6.8   0:27.20 ffmpeg
  109 root      20   0       0      0      0 I  24.5   0.0   0:08.26 kworker/u2:2-brcmf_wq/mmc1:0001:1
    3 root      20   0       0      0      0 I   5.2   0.0   0:01.97 kworker/0:0-events
  505 pi        20   0   63116   2464   1860 S   1.6   0.7   0:00.95 raspivid

# YouTube Live Stream
The YouTube Live streaming service lets anyone with a YouTube account stream live video from a device with a video camera that can output h264 video and stream it via RTMP. 


## First Attempt
My first attempt worked.  But there was too much buffering.  The YouTube UI says:
```
Error
YouTube is not receiving enough video to maintain smooth streaming. As such, viewers will experience buffering.

```
My setup was based on [Digikey's article](https://www.digikey.com/en/maker/blogs/streaming-live-to-youtube-and-facebook-using-raspberry-pi-camera).  

The gobley gook command the article recommends:
```
raspivid -o - -t 0 -vf -hf -fps 10 -b 500000 | ffmpeg -re -ar 44100 -ac 2 -acodec pcm_s16le -f s16le -ac 2 -i /dev/zero -f h264 -i - -vcodec copy -acodec aac -ab 128k -g 50 -strict experimental -f flv rtmp://a.rtmp.youtube.com/live2/STREAM-KEY
```
The Rasp Pi was gave a message:
```
[h264 @ 0x348a1e0] Thread message queue blocking; consider raising the thread_queue_size option (current value: 8)
[flv @ 0x34a56e0] Timestamps are unset in a packet for stream 0. This is deprecated and will stop working in the future. Fix your code to set the timestamps properly


frame= 8862 fps= 10 q=-1.0 size=   54608kB time=00:05:54.47 bitrate=1262.0kbits/s speed=0.406x
```
Then eventually failed:
```
[flv @ 0x34a56e0] Failed to update header with correct duration.
[flv @ 0x34a56e0] Failed to update header with correct filesize.
Error writing trailer of rtmp://a.rtmp.youtube.com/live2/bucf-ah5k-p9rr-bvf8-1xjk: End of file
frame=43501 fps= 10 q=-1.0 Lsize=  268102kB time=00:29:00.03 bitrate=1262.2kbits/s speed=0.401x
video:265565kB audio:443kB subtitle:0kB other streams:0kB global headers:0kB muxing overhead: 0.787225%
[aac @ 0x34c2fd0] Qavg: 65536.000
Conversion failed!
```
OH MY!...before I begin to understand all that's going on here, I'll try an updated raspivid command
```
raspivid -o - -t 0 -vf -hf -fps 30 -b 6000000 | ffmpeg -re -ar 44100 -ac 2 -acodec pcm_s16le -f s16le -ac 2 -i /dev/zero -f h264 -i - -vcodec copy -acodec aac -ab 128k -g 50 -strict experimental -f flv rtmp://a.rtmp.youtube.com/live2/[your-secret-key-here]
```
### Raspivid commands
(see the [Rasp Pi doc for the camera](https://www.raspberrypi.org/documentation/raspbian/applications/camera.md) for more info on the commands)
The commands above use:
- -o for output filename.  Both seem to use - which I assume means no name.
- -t 0 for running indefinately.
- -vf and -hf for vertical and horizontal flip.  I guess the rasp pi camera needs this.  I don't know.
- -fps for Frames Per Second to record.  
- -b for bitrates per second.
The difference seems to be in frames per second

```
raspivid -o - -t 0 -w 1920 -h 1080 -fps 25 -b 3000000 -rot 270 -g 50 -f | ffmpeg -thread_queue_size 10240 -f h264 -r 25 -i - -itsoffset 5.3 -f alsa -thread_queue_size 10240 -ac 2 -i hw:1,0 -vcodec copy -acodec aac -ac 2 -ar 44100 -ab 192k -f flv rtmp://a.rtmp.youtube.com/live2/YOUR_STREAM_KEY
```
- thread_queue_size should help with the error....
- -f is for the file format - h264
- -r 25 says to force the fps to 25. 
- -itsoffset sets input time offset.  I don't know what 5.3 is...
- -f alsa is for grabbing audio - which I don't have at this time.  So removing, we get:
```
raspivid -o - -t 0 -w 1920 -h 1080 -fps 25 -b 3000000 -rot 270 -g 50 -f | ffmpeg -thread_queue_size 10240 -f h264 -r 25 -i - -itsoffset 5.3  -vcodec copy -acodec aac -ac 2 -ar 44100 -ab 192k -f flv rtmp://a.rtmp.youtube.com/live2/YOUR_STREAM_KEY
```
Running the above gave me the error:
```
Option itsoffset (set the input ts offset) cannot be applied to output url rtmp://a.rtmp.youtube.com/live2/bucf-ah5k-p9rr-bvf8-1xjk -- you are trying to apply an input option to an output file or vice versa. Move this option before the file it belongs to.
Error parsing options for output file rtmp://a.rtmp.youtube.com/live2/bucf-ah5k-p9rr-bvf8-1xjk.
Error opening output files: Invalid argument
```
So removing that:
```
raspivid -o - -t 0 -w 1920 -h 1080 -fps 25 -b 3000000 -rot 270 -g 50 -f | ffmpeg -thread_queue_size 10240 -f h264 -r 25 -i -   -vcodec copy -acodec aac -ac 2 -ar 44100 -ab 192k -f flv rtmp://a.rtmp.youtube.com/live2/YOUR_STREAM_KEY
```
Now I get:
```
Codec AVOption b (set bitrate (in bits/s)) specified for output file #0 (rtmp://a.rtmp.youtube.com/live2/bucf-ah5k-p9rr-bvf8-1xjk) has not been used for any stream. The most likely reason is either wrong type (e.g. a video option with no video streams) or that it is a private option of some encoder which was not actually used for any stream.
```
and while the stream seems to be pumping out of the Rasp Pi, its not showing up on YouTube...

My new try will be:
```
raspivid -o - -t 0 -vf -hf -fps 25 -b 3000000 | ffmpeg -thread_queue_size 10240 -f h264 -r 25 -i - -vcodec copy -acodec aac -ab 128k -g 50 -strict experimental -f flv rtmp://a.rtmp.youtube.com/live2/[your-secret-key-here]
```

raspivid -o - -t 0 -vf -hf -fps 25 -b 3000000 | ffmpeg -thread_queue_size 10240 -r 25 -re -ar 44100 -ac 2 -acodec pcm_s16le -f s16le -ac 2 -i /dev/zero -f h264 -i - -vcodec copy -acodec aac -ab 128k -g 50 -strict experimental -f flv rtmp://a.rtmp.youtube.com/live2/[your-secret-key-here]

# DEBUG AREA
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
- TODO: Update this part after current experiment ends.

- got to 1 hr 52 minutes before ssh command line stopped streaming.  Not sure how to debug.
```
client_loop: send disconnect: Connection reset by peer:48.99 bitrate= 844.6kbits/s speed= 1.2x
```
have to reboot rasp pi.
chip feels hot -> need heat sink?

- now try cronjob.
- put in heatsink

lasted 12 hrs 49 mins.  The GOOD news is I can get to rasp pi.
## Debug ffmpeg
I could give up at this point, but in the quest of learning - how can I debug `ffmpeg` and what do I get if I do.
### -loglevel debug
The command then becomes:
```
raspivid -o - -t 0 -b 1000000 | ffmpeg -i - -f s16le -i /dev/zero -c:v copy -c:a aac -g 50 -f flv -flvflags no_duration_filesize rtmp://a.rtmp.youtube.com/live2/[YOUTUBE LIVE KEY]
```
FFREPORT=file=ffreport.log:level=48

raspivid -o - -t 0 -b 1000000 | FFREPORT="file=ffreport.log:level=48" ffmpeg -i - -f s16le -i /dev/zero -c:v copy -c:a aac -g 50 -f flv -flvflags no_duration_filesize rtmp://a.rtmp.youtube.com/live2/[YOUTUBE LIVE KEY]
```

[adding -ih to raspivid]()
raspivid -o - -ih -t 0 -b 1000000 | FFREPORT="file=ffreport.log:level=48" ffmpeg -i - -f s16le -i /dev/zero -c:v copy -c:a aac -g 50 -f flv -flvflags no_duration_filesize rtmp://a.rtmp.youtube.com/live2/[YOUTUBE LIVE KEY]

![htop when logging turned on w/ ffmpeg](images\htop_ffmpeg_after_18hours.jpg)

```
ffmpeg started on 2021-06-06 at 05:57:40
Report written to "ffreport.log"
Command line:
ffmpeg -i - -f s16le -i /dev/zero -c:v copy -c:a aac -g 50 -f flv -flvflags no_duration_filesize rtmp://a.rtmp.youtube.com/live2/bucf-ah5k-p9rr-bvf8-1xjk
ffmpeg version 4.1.6-1~deb10u1+rpt1 Copyright (c) 2000-2020 the FFmpeg developers
  built with gcc 8 (Raspbian 8.3.0-6+rpi1)
  configuration: --prefix=/usr --extra-version='1~deb10u1+rpt1' --toolchain=hardened --incdir=/usr/include/arm-linux-gnueabihf --enable-gpl --disable-stripping --enable-avresample --disable-filter=resample --enable-avisynth --enable-gnutls --enable-ladspa --enable-libaom --enable-libass --enable-libbluray --enable-libbs2b --enable-libcaca --enable-libcdio --enable-libcodec2 --enable-libflite --enable-libfontconfig --enable-libfreetype --enable-libfribidi --enable-libgme --enable-libgsm --enable-libjack --enable-libmp3lame --enable-libmysofa --enable-libopenjpeg --enable-libopenmpt --enable-libopus --enable-libpulse --enable-librsvg --enable-librubberband --enable-libshine --enable-libsnappy --enable-libsoxr --enable-libspeex --enable-libssh --enable-libtheora --enable-libtwolame --enable-libvidstab --enable-libvorbis --enable-libvpx --enable-libwavpack --enable-libwebp --enable-libx265 --enable-libxml2 --enable-libxvid --enable-libzmq --enable-libzvbi --enable-lv2 --enable-omx --enable-openal --enable-opengl  libavutil      56. 22.100 / 56. 22.100
  libavcodec     58. 35.100 / 58. 35.100
  libavformat    58. 20.100 / 58. 20.100
  libavdevice    58.  5.100 / 58.  5.100
  libavfilter     7. 40.101 /  7. 40.101
  libavresample   4.  0.  0 /  4.  0.  0
  libswscale      5.  3.100 /  5.  3.100
  libswresample   3.  3.100 /  3.  3.100
  libpostproc    55.  3.100 / 55.  3.100
Splitting the commandline.
Reading option '-i' ... matched as input url with argument '-'.
Reading option '-f' ... matched as option 'f' (force format) with argument 's16le'.
Reading option '-i' ... matched as input url with argument '/dev/zero'.
Reading option '-c:v' ... matched as option 'c' (codec name) with argument 'copy'.
Reading option '-c:a' ... matched as option 'c' (codec name) with argument 'aac'.
Reading option '-g' ... matched as AVOption 'g' with argument '50'.
Reading option '-f' ... matched as option 'f' (force format) with argument 'flv'.
Reading option '-flvflags' ... matched as AVOption 'flvflags' with argument 'no_duration_filesize'.
Reading option 'rtmp://a.rtmp.youtube.com/live2/bucf-ah5k-p9rr-bvf8-1xjk' ... matched as output url.
Finished splitting the commandline.
Parsing a group of options: global .
Successfully parsed a group of options.
Parsing a group of options: input url -.
Successfully parsed a group of options.
Opening an input file: -.
[NULL @ 0x1f1e070] Opening 'pipe:' for reading
[pipe @ 0x1f1e800] Setting default whitelist 'crypto'
[h264 @ 0x1f1e070] Format h264 probed with size=2048 and score=51
[h264 @ 0x1f1e070] Before avformat_find_stream_info() pos: 0 bytes read:32768 seeks:0 nb_streams:1
[AVBSFContext @ 0x1f42c70] nal_unit_type: 7(SPS), nal_ref_idc: 1
[AVBSFContext @ 0x1f42c70] nal_unit_type: 8(PPS), nal_ref_idc: 1
[AVBSFContext @ 0x1f42c70] nal_unit_type: 5(IDR), nal_ref_idc: 1
[h264 @ 0x1f20ae0] nal_unit_type: 7(SPS), nal_ref_idc: 1
[h264 @ 0x1f20ae0] nal_unit_type: 8(PPS), nal_ref_idc: 1
[h264 @ 0x1f20ae0] nal_unit_type: 5(IDR), nal_ref_idc: 1
[h264 @ 0x1f20ae0] Format yuv420p chosen by get_format().
[h264 @ 0x1f20ae0] Reinit context to 1920x1088, pix_fmt: yuv420p
[h264 @ 0x1f20ae0] nal_unit_type: 1(Coded slice of a non-IDR picture), nal_ref_idc: 1
[h264 @ 0x1f20ae0] nal_unit_type: 1(Coded slice of a non-IDR picture), nal_ref_idc: 1
[h264 @ 0x1f20ae0] nal_unit_type: 1(Coded slice of a non-IDR picture), nal_ref_idc: 1
[h264 @ 0x1f20ae0] nal_unit_type: 1(Coded slice of a non-IDR picture), nal_ref_idc: 1
[h264 @ 0x1f20ae0] nal_unit_type: 1(Coded slice of a non-IDR picture), nal_ref_idc: 1
[h264 @ 0x1f20ae0] nal_unit_type: 1(Coded slice of a non-IDR picture), nal_ref_idc: 1
[h264 @ 0x1f1e070] max_analyze_duration 5000000 reached at 5000000 microseconds st:0
[h264 @ 0x1f1e070] After avformat_find_stream_info() pos: 557056 bytes read:557056 seeks:0 frames:127
Input #0, h264, from 'pipe:':
  Duration: N/A, bitrate: N/A
    Stream #0:0, 127, 1/1200000: Video: h264 (High), yuv420p(progressive), 1920x1080, 25 fps, 25 tbr, 1200k tbn, 50 tbc
Successfully opened the file.
Parsing a group of options: input url /dev/zero.
Applying option f (force format) with argument s16le.
Successfully parsed a group of options.
Opening an input file: /dev/zero.
[s16le @ 0x1f40e20] Opening '/dev/zero' for reading
[file @ 0x1fcf890] Setting default whitelist 'file,crypto'
[s16le @ 0x1f40e20] Before avformat_find_stream_info() pos: 0 bytes read:32768 seeks:0 nb_streams:1
[s16le @ 0x1f40e20] All info found
[s16le @ 0x1f40e20] After avformat_find_stream_info() pos: 102400 bytes read:131072 seeks:0 frames:50
Guessed Channel Layout for Input Stream #1.0 : mono
Input #1, s16le, from '/dev/zero':
  Duration: N/A, bitrate: 705 kb/s
    Stream #1:0, 50, 1/44100: Audio: pcm_s16le, 44100 Hz, mono, s16, 705 kb/s
Successfully opened the file.
Parsing a group of options: output url rtmp://a.rtmp.youtube.com/live2/bucf-ah5k-p9rr-bvf8-1xjk.
Applying option c:v (codec name) with argument copy.
Applying option c:a (codec name) with argument aac.
Applying option f (force format) with argument flv.
Successfully parsed a group of options.
Opening an output file: rtmp://a.rtmp.youtube.com/live2/bucf-ah5k-p9rr-bvf8-1xjk.
[rtmp @ 0x1fd7910] No default whitelist set
[tcp @ 0x1fd7e20] No default whitelist set
[tcp @ 0x1fd7e20] Original list of addresses:
[tcp @ 0x1fd7e20] Address 142.250.217.76 port 1935
[tcp @ 0x1fd7e20] Address 142.250.217.108 port 1935
[tcp @ 0x1fd7e20] Address 172.217.14.204 port 1935
[tcp @ 0x1fd7e20] Address 172.217.14.236 port 1935
[tcp @ 0x1fd7e20] Address 142.250.69.204 port 1935
[tcp @ 0x1fd7e20] Address 142.251.33.108 port 1935
[tcp @ 0x1fd7e20] Address 142.251.33.76 port 1935
[tcp @ 0x1fd7e20] Address 2607:f8b0:400a:80a::200c port 1935
[tcp @ 0x1fd7e20] Address 2607:f8b0:400a:80b::200c port 1935
[tcp @ 0x1fd7e20] Address 2607:f8b0:400a:801::200c port 1935
[tcp @ 0x1fd7e20] Address 2607:f8b0:400a:803::200c port 1935
[tcp @ 0x1fd7e20] Interleaved list of addresses:
[tcp @ 0x1fd7e20] Address 142.250.217.76 port 1935
[tcp @ 0x1fd7e20] Address 2607:f8b0:400a:80a::200c port 1935
[tcp @ 0x1fd7e20] Address 142.250.217.108 port 1935
[tcp @ 0x1fd7e20] Address 2607:f8b0:400a:80b::200c port 1935
[tcp @ 0x1fd7e20] Address 172.217.14.204 port 1935
[tcp @ 0x1fd7e20] Address 2607:f8b0:400a:801::200c port 1935
[tcp @ 0x1fd7e20] Address 172.217.14.236 port 1935
[tcp @ 0x1fd7e20] Address 2607:f8b0:400a:803::200c port 1935
[tcp @ 0x1fd7e20] Address 142.250.69.204 port 1935
[tcp @ 0x1fd7e20] Address 142.251.33.108 port 1935
[tcp @ 0x1fd7e20] Address 142.251.33.76 port 1935
[tcp @ 0x1fd7e20] Starting connection attempt to 142.250.217.76 port 1935
[tcp @ 0x1fd7e20] Successfully connected to 142.250.217.76 port 1935
[rtmp @ 0x1fd7910] Handshaking...
[rtmp @ 0x1fd7910] Type answer 3
[rtmp @ 0x1fd7910] Server version 4.0.0.1
[rtmp @ 0x1fd7910] Proto = rtmp, path = /live2/bucf-ah5k-p9rr-bvf8-1xjk, app = live2, fname = bucf-ah5k-p9rr-bvf8-1xjk
[rtmp @ 0x1fd7910] Window acknowledgement size = 2500000
[rtmp @ 0x1fd7910] Max sent, unacked = 10000000
[rtmp @ 0x1fd7910] New incoming chunk size = 256
[rtmp @ 0x1fd7910] Releasing stream...
[rtmp @ 0x1fd7910] FCPublish stream...
[rtmp @ 0x1fd7910] Creating stream...
[rtmp @ 0x1fd7910] Sending publish command for 'bucf-ah5k-p9rr-bvf8-1xjk'
Successfully opened the file.
Stream mapping:
  Stream #0:0 -> #0:0 (copy)
  Stream #1:0 -> #0:1 (pcm_s16le (native) -> aac (native))
cur_dts is invalid (this is harmless if it occurs once at the start per stream)
cur_dts is invalid (this is harmless if it occurs once at the start per stream)
detected 1 logical cores
[graph_0_in_1_0 @ 0x1fa2e80] Setting 'time_base' to value '1/44100'
[graph_0_in_1_0 @ 0x1fa2e80] Setting 'sample_rate' to value '44100'
[graph_0_in_1_0 @ 0x1fa2e80] Setting 'sample_fmt' to value 's16'
[graph_0_in_1_0 @ 0x1fa2e80] Setting 'channel_layout' to value '0x4'
[graph_0_in_1_0 @ 0x1fa2e80] tb:1/44100 samplefmt:s16 samplerate:44100 chlayout:0x4
[format_out_0_1 @ 0x1fa2a30] Setting 'sample_fmts' to value 'fltp'
[format_out_0_1 @ 0x1fa2a30] Setting 'sample_rates' to value '96000|88200|64000|48000|44100|32000|24000|22050|16000|12000|11025|8000|7350'
[format_out_0_1 @ 0x1fa2a30] auto-inserting filter 'auto_resampler_0' between the filter 'Parsed_anull_0' and the filter 'format_out_0_1'
[AVFilterGraph @ 0x1fa2ac0] query_formats: 4 queried, 6 merged, 3 already done, 0 delayed
[auto_resampler_0 @ 0x1fa3ca0] [SWR @ 0x1fb41b0] Using s16p internally between filters
[auto_resampler_0 @ 0x1fa3ca0] ch:1 chl:mono fmt:s16 r:44100Hz -> ch:1 chl:mono fmt:fltp r:44100Hz
Output #0, flv, to 'rtmp://a.rtmp.youtube.com/live2/bucf-ah5k-p9rr-bvf8-1xjk':
  Metadata:
    encoder         : Lavf58.20.100
    Stream #0:0, 0, 1/1000: Video: h264 (High) ([7][0][0][0] / 0x0007), yuv420p(progressive), 1920x1080, q=2-31, 25 fps, 25 tbr, 1k tbn, 1200k tbc
    Stream #0:1, 0, 1/1000: Audio: aac (LC) ([10][0][0][0] / 0x000A), 44100 Hz, mono, fltp, 69 kb/s
    Metadata:
      encoder         : Lavc58.35.100 aac
cur_dts is invalid (this is harmless if it occurs once at the start per stream)
cur_dts is invalid (this is harmless if it occurs once at the start per stream)
[h264 @ 0x1f1e070] Thread message queue blocking; consider raising the thread_queue_size option (current value: 8)
[flv @ 0x1fd4c80] Timestamps are unset in a packet for stream 0. This is deprecated and will stop working in the future. Fix your code to set the timestamps properly
cur_dts is invalid (this is harmless if it occurs once at the start per stream)
frame=    5 fps=0.0 q=-1.0 size=      21kB time=00:00:00.16 bitrate=1076.0kbits/s speed=0.317x    
frame=   13 fps= 13 q=-1.0 size=      53kB time=00:00:00.51 bitrate= 842.7kbits/s speed=0.504x    
frame=   26 fps= 17 q=-1.0 size=      92kB time=00:00:01.04 bitrate= 722.6kbits/s speed=0.688x    
frame=   40 fps= 20 q=-1.0 size=     145kB time=00:00:01.56 bitrate= 759.4kbits/s speed=0.766x    
frame=   62 fps= 24 q=-1.0 size=     267kB time=00:00:02.44 bitrate= 896.3kbits/s speed=0.951x    
frame=   88 fps= 28 q=-1.0 size=     356kB time=00:00:03.48 bitrate= 837.4kbits/s speed=1.13x    
frame=  107 fps= 30 q=-1.0 size=     435kB time=00:00:04.24 bitrate= 839.8kbits/s speed=1.17x    
frame=  122 fps= 29 q=-1.0 size=     527kB time=00:00:04.84 bitrate= 892.3kbits/s speed=1.17x    
frame=  147 fps= 32 q=-1.0 size=     601kB time=00:00:05.85 bitrate= 841.3kbits/s speed=1.26x    
frame=  170 fps= 33 q=-1.0 size=     701kB time=00:00:06.80 bitrate= 843.5kbits/s speed=1.32x    
frame=  192 fps= 34 q=-1.0 size=     804kB time=00:00:07.68 bitrate= 856.6kbits/s speed=1.36x    
frame=  214 fps= 35 q=-1.0 size=     879kB time=00:00:08.52 bitrate= 845.4kbits/s speed=1.39x    
frame=  237 fps= 36 q=-1.0 size=     972kB time=00:00:09.44 bitrate= 843.5kbits/s speed=1.42x    
frame=  262 fps= 37 q=-1.0 size=    1077kB time=00:00:10.44 bitrate= 844.8kbits/s speed=1.46x    
frame=  286 fps= 37 q=-1.0 size=    1172kB time=00:00:11.40 bitrate= 842.4kbits/s speed=1.48x    
frame=  299 fps= 36 q=-1.0 size=    1229kB time=00:00:11.92 bitrate= 844.5kbits/s speed=1.46x    
frame=  332 fps= 38 q=-1.0 size=    1365kB time=00:00:13.24 bitrate= 844.4kbits/s speed=1.52x    
frame=  373 fps= 41 q=-1.0 size=    1547kB time=00:00:14.88 bitrate= 851.4kbits/s speed=1.62x    
frame=  416 fps= 43 q=-1.0 size=    1713kB time=00:00:16.62 bitrate= 844.0kbits/s speed=1.71x    
frame=  460 fps= 45 q=-1.0 size=    1891kB time=00:00:18.36 bitrate= 843.5kbits/s speed= 1.8x    
frame=  501 fps= 47 q=-1.0 size=    2063kB time=00:00:20.00 bitrate= 844.9kbits/s speed=1.87x    
frame=  542 fps= 48 q=-1.0 size=    2260kB time=00:00:21.64 bitrate= 855.7kbits/s speed=1.93x    
frame=  588 fps= 50 q=-1.0 size=    2422kB time=00:00:23.48 bitrate= 845.1kbits/s speed=2.01x    
frame=  617 fps= 51 q=-1.0 size=    2546kB time=00:00:24.64 bitrate= 846.5kbits/s speed=2.02x    
frame=  642 fps= 50 q=-1.0 size=    2642kB time=00:00:25.64 bitrate= 843.9kbits/s speed=2.01x    
.
.
.
frame=1673051 fps= 30 q=-1.0 size= 6868813kB time=18:35:22.04 bitrate= 840.8kbits/s speed= 1.2x    
frame=1673065 fps= 30 q=-1.0 size= 6868823kB time=18:35:22.56 bitrate= 840.8kbits/s speed= 1.2x    
.
.
.
frame=2627628 fps= 30 q=-1.0 size= 7951326kB time=29:11:45.08 bitrate= 619.7kbits/s speed= 1.2x    
frame=2627658 fps= 30 q=-1.0 size= 7951459kB time=29:11:46.30 bitrate= 619.7kbits/s speed= 1.2x    
frame=2627672 fps= 30 q=-1.0 size= 7951514kB time=29:11:46.86 bitrate= 619.7kbits/s speed= 1.2x    

```

from a reddit ? about why bitrate dropped in the above:
```
FFmpeg's reported bitrate is just the size/time. This is an overall average that will change less the longer a stream is run. If your stream has a lot of easy to encode frames (low motion or black screen) it could cause encoder to be unable to get up to the target bitrate even at its lowest QP.

You can use VLC to view a section of your stream after it has been going for a while. The current bitrate plot can be viewed in Tools>Codec Information>Statistics>Input/Read>Input Bitrate
```
