+++
title = "Teensy Audio Streaming with JackTrip"
date = "2022-04-03T20:07:47+02:00"

#
# description is optional
#
# description = "An optional description for SEO. If not provided, an automatically created summary will be used."

tags = ["teensy","jacktrip"]
+++

Get the code on [GitHub](https://github.com/msmd-io/TeensyAudioStreaming).

I've written some code that lets you use your Teensy 4.1 as a [JackTrip](https://ccrma.stanford.edu/software/jacktrip/) client using a Teensy 4.1 + Audio Shield + Ethernet connector.  You can optionally connect a small colour LED display to show some info such as the Teensy's IP Address and whether it's receiving/transmitting data.

The remote JackTrip device's IP address must be input using the serial monitor. However, this IP address is stored in the EPROM of the Teensy so if the IP address is static you only have to set it once and it will autoconnect after 10 seconds on future boots.

Anything that is connected to the line input or USB Teensy Soundcard input is mixed together and sent to the remote jacktrip server/client via the ethernet connection.

Anything that is received via the ethernet connection is send to the line output and the USB Teensy Soundcard ouput.

There is a variable that can be set in the code to enable audio monitoring/loopback of the input signal.
