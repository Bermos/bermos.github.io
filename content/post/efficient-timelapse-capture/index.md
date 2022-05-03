+++
author = "Matthew Machivenyika (Bermos)"
title = "Efficiently capture timelapses"
date = "2022-05-03"
description = "How to reduce storage space used while capturing timelapses over long durations."
categories = [
    "General Knowledge"
]
tags = [
    "timelapse",
    "photo",
    "video",
    "storage",
    "ffmpeg"
]
+++

# Efficiently capture timelapses

## The problem

You want to capture a timelapse over the course of lets say a week.
For reference, I'm going to use the defaults given by the RaspberryPi camera module.
If any kind of detail of movement should be captured the length between taking single pictures cannot be too great.
For active scenes with people moving about a picture once every minute won't cut it.
One every 5 seconds is from my personal testing about right to capture human scale movement.

Pictures taken by the RaspberryPi camera module are about 5 MP in size, in .jpeg that comes out to about 2.4MB per picture [^1].
For a week of timelapsing that's a whooping `2.4 * 12 * 60 * 24 * 7 = 290304MB = 283.5GB`! Even with SD card prices
dropping as they have it is still a rather costly endeavour.
[^1]: "Raspberry Pi Documentation - Camera", https://www.raspberrypi.com/documentation/accessories/camera.html#file-size


## The idea

Now most timelapses have a rather neat property. Most of the time, most of the picture won't change from frame to frame.
This means a lot of the data is redundant! It would therefore make sense if we could only store the *usually* small
difference between each frame.


## The (a) solution

The video codec H.264 and it's newer iteration H.265 do exactly that. They create keyframes which are full pictures and
every subsequent frame is calculated by its difference to its predecessor. Which also means H.26x only has to store
keyframes + the changes in each frame. A test with the less efficient codec H.264 resulted in about 64GB of storage
used for 120 continuous days of taking one picture every 5 seconds. Of course the exact storage consumption will vary
depending on how much each picture differs from the next.


## Examples

Sounds cool and whatnot, how do I actually encode a video on the fly?
The answer as with all things AV is of course [ffmpeg](https://www.ffmpeg.org/).

`ffmpeg -f image2pipe -framerate 25 -i - -c:v libx264 -f mp4 -r 25 output.mp4`

Let me explain what this command is about.
- `-f image2pipe`: force input to be the image2pipe format
- `-framerate 25`: this is purely personal, if you want your timelapse to have 60fps set it here
- `-i -`: get the image from the pipe `-`, this will be useful shortly
- `-c:v libx264`: use the H.264 codec
- `-f mp4`: force output to be in the mp4 format
- `-r 25`: force the output video framerate to 25fps
- `output.mp4`: output file name

So to recap this command instructs ffmpeg to take images from the pipe and
encode them into the output.mp4 file as long as the pipe is open.
You can use named pipes to keep this command open and pipe the images into it when they are taken.


## It can't be that easy, can it?

Of course not. Your Raspberry might crash, lose power, or you accidentally stop the image
taking or the ffmpeg process. If that happens you'll find that your video is corrupted.
It can be rescued if you are lucky, but I wouldn't count on it.

`ffmpeg -f image2pipe -framerate 25 -i - -g 120 -c:v libx264 -f mp4 -movflags frag_keyframe+empty_moov -r 25 args.out_name`

So what does this extended ffmpeg command do?
- `-g 120`: sets the interval between keyframes to 120. So at max 119 frames can be lost. Decreasing the number minimises the loss but increases the file size.
- `-movflags frag_keyframe+empty_moov`: this moves the movie encoding information to the beginning of the file. Normally this would be written at the end when
  the ffmpeg process is terminated gracefully.

These modifications can confuse some lesser video players so after recording is finished it
is recommended to re-encode the video with this `ffmpeg -i input.mp4 -c copy output.mp4` easy
command.

## The code

So, after many words what was actually done with the knowledge gained?

I wrote a Python script that would let us capture the LAN party we organized with our student
association from setting up the first table to cleaning the last RedBull can.

Code: [Raspberry PI timelapse recoder](https://github.com/VSETH-GECO/PiCam)