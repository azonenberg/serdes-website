---
layout: post
title:  "Embedded infrastructure, part 2: Power"
date:   2024-07-17 09:00:00 -0700
---

## The problem

I started out building tiny little USB 2.0 powered FPGA/MCU dev boards that didn't even get warm to the touch. But the
power budget of something like a 48-port Ethernet switch, no matter how efficient you make it, is going to vastly
exceed the 2.5W I can pull from a passive USB wallwart. The trigger crossbar prototype on my bench now is pulling just
shy of 10W and the switch will will need ~30W for the 4x 12-port baseT line cards plus whatever the FPGA based switch
engine and management system uses.

I've used 5/12V barrel jacks for some stuff but they have an annoying tendency to come loose and aren't the best for
higher power levels anyway.

A direct mains inlet is an option, but comes with more annoying safety concerns (and regulations if I want to make
these for anyone but me to use). So that's not my first choice.

USB PD is the trendy option these days, but requires a fairly smart power brick for each device. I am planning to put a
bunch of these gizmos around my lab and if I could run them off a common power supply that would be nice given my
limited mains power capacity.

## The plan

There's another option popular in the datacenter/telecom world: 48V DC. I decided to go with this route - specifically
+48V DC on a six pin Molex Mini-Fit Jr connector with the negative side at earth ground potential, since there's
commercially available power bricks that supply this but it's also straightforward to adapt to e.g. a rack wide DC bus
system in the future if I wanted to go that route.

48V is a bit high to feed directly to most of the DC-DC modules I use for my projects, so I designed a reusable
intermediate bus converter to step the 48V input down to 12V on an 8-pin Mini-Fit Jr. The IBC also provides an
always-on 3.3V standby rail to power supervisor/management logic and a bunch of I2C sensors.

## Current state

The first generation IBC, after a board spin to fix a stupid layout error that slipped through design review, is
functional but has annoyingly high standby power consumption (around 3W) causing awful efficiency at light loads.

I've got a second generation of the IBC at fab now that's about 40% smaller PCB wise and should have idle power closer
to 0.5W, greatly improving low-load efficiency.
