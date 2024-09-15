---
title: "Atmospheric Balloon Payload Competition 2015"
description: "A competition during my bachelor study"
date: 2021-01-11T08:00:00
draft: False
tags: ["projects"]
---


One of my first exposure to remote sensing and Python programming. Held anually by the National Institute of Aeronautics and Space (LAPAN), the Atmospheric Balloon Payload Competition challenges students to design a payload attached to a weather baloon to measure atmospheric conditions while maintaining active communication to a ground station. We participate in a team of three, I was the captain and in charge of sensors and programming.

Summary of hardware:
- Raspberry Pi model A with Raspberry Pi Camera
- GPS: location and altitude for antenna tracker
- 3-axis gyroscope, accelerometer, and compass: for windspeed and direction
- barometer: atmospheric pressure
- temperature and humidity sensor
- 433MHz telemetry radio module for communication


We were newcomers coming outside of the Java island (where top universities are). Long story short, the GUI I was so proud of kept crashing due to tranmsission errors (most likely interference from other payloads), creating cryptic bytes that were not handled. We had to parse the received data manually and the judges were not happy about it. Lessons learned: prepare cases and catch errors. It was such a memorable experience though!

![team + our mentor](/projects/kombat-2015/team.jpg)

I was so stressed out during preparation (we came from outside the land, so our huge antenna needs to be reassembled and tested) that I forgot to take pictures of our payload and a proper team photo during the competition.