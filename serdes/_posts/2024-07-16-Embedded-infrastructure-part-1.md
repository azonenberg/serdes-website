---
layout: post
title:  "Embedded infrastructure, part 1: Goals"
date:   2024-07-16 21:00:00 -0700
---

To kick off my new blog I thought I'd start a series on the embedded platform I've been building over the past year and
change as part of my various projects. This was going to just be one post but it got long enough I'm breaking it up...

I've been working towards a bunch of large embedded hardware projects for a long time - a 25 Gbps BERT, a 48+2 port
1/10 gigabit Ethernet switch, and a 2 GHz oscilloscope are all on the menu, with a few smaller projects in progress (or
completed) as stepping stones (trigger crossbar, 14+1 port 1/10G switch, etc).

This is an introductory series to the whole family of projects so there won't be a lot of technical detail here, just
setting the stage for what I'm doing and why, and a quick summary of where things stand now. If you don't follow my
socials and aren't familiar with the projects already, that's fine - I'll be posting more about them when the time
comes.

## So what needs doing?

All of these projects have a lot in common: they're going to be 1U rackmounts with IP management via SSH and/or SCPI
and a bunch of front panel ports whose function will vary from device to device. The back panel will need to provide
power, management network, serial console for debug/provisioning, and maybe some auxiliary ports like reference clock
and trigger synchronization.

Internally, there will need to be a decent-sized FPGA handling the high speed datapath and IO as well as a
microcontroller running the management interface, and they have to talk somehow. There's also going to be a lot of
power domains and reset signals to wrangle, so I need some glue to control all of that.

And the whole thing needs to be powered somehow.
