---
title: Show a dashboard at login on tty1
tags: [linux, docker]                   
date: 2019-09-08T21:21:12+01:00
---

Recently I wanted to change the default login prompt on the tty1 console on a bare-metal instance to automatically run dockli, a command-line based docker dashboard. 
<!--more--> 

# About TTY1, getty and agetty
In the 1830s and 1840s, machines known as teleprinters were developed. These machines could send typed messages 
“down the wire” to distant locations. The messages were typed by the sender on a keyboard of sorts. 
They were printed on paper at the receiving end. They were an evolutionary step in telegraphy, which had previously relied on Morse 
and similar codes.

Messages were encoded and transmitted, then received, decoded, and printed. 
There were several techniques used to encode and decode the messages. 
The most famous, and one of the most prolific, was patented in 1874 by Émile Baudot, for whom the baud rate is named. 
His character encoding scheme pre-dated ASCII by 89 years.

Each "tty" device was connected via a serial port, because copper wire was expensive and so "parallel" port devices were mainly used for 
short-distance interfaces like to a local printer. 
This is before wireless networks were widely used. In the old days of multi-user computer environments you could have multiple "terminal devices",
 i.e., 'tty' devices connected to the same central computer. 
 This is the original hardware environment in which Unix was developed. 
 That hardware legacy is still with us in the naming of the software components in the Linux operating system.

Today in Linux, the tty is a legacy name used to refer to the user interface for text-based input and output, otherwise known as a "terminal". 
In Linux systems, there can be multiple tty device "consoles", to support potentially dozens of serial ports or more. 
`tty0` is the current one in use, but Linux allows you to switch to another session by changing to a different tty, e.g., `tty1`. 
Linux supports up to 6 ttys by default, but this number is configurable.

In practical terms, think of a tty as a serial communication channel that a Linux session uses to communicate with a user.


> Interested in the history of TTY? Read this [great post](http://www.linusakesson.net/programming/tty/) from  **lft**

A [getty](https://en.wikipedia.org/wiki/Getty_(Unix)) is the generic name for a program which manages a terminal line and its connected terminal. 
Its purpose is to protect the system from unauthorized access. 
Generally, each getty process is started by [systemd](https://wiki.archlinux.org/index.php/Systemd) and manages a single terminal line.

# DRY 
Dry is a terminal application to manage Docker containers and images. 
It aims to be an alternative to the official Docker CLI and, more interesting for us,as a tool to monitor Docker containers from a terminal.

You can think of Dry as htop for containers.
{% include blog/figure.html img="dry.png" alt="DRY htop for docker" caption="DRY, htop for containers!" %}

# Start dry instead of login prompt
The GOAL is to have `DRY` as a default screen, instead of the login prompt, on `tty0`. We will still be able to log in using another tty (there are 6 by default!).
 
## Configure tty1
We need to create a systemd override to use input/output redirection on tty1 
```bash
mkdir /etc/systemd/system/getty@tty1.service.d/
vim /etc/systemd/system/getty@tty1.service.d/override.conf
```

Paste the following:
```
[Service]
ExecStart=
ExecStart=-/bin/bash -c "/usr/bin/docker run --rm -it -v /var/run/docker.sock:/var/run/docker.sock -e DOCKER_HOST=$DOCKER_HOST moncho/dry"
StandardInput=tty
StandardOutput=tty
```

That's it! Now reload the unit files and restart the service. `DRY` should appear on the prompt
```
systemctl daemon-reload; systemctl restart getty@tty1.service
``` 

Or do a simple `reboot`.

That's it! Now our servers show some hinsight about our containers instead of an anonmous login prompt!