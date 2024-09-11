---
layout: post
title:  "Intermediate Bus Converter"
date:   2024-09-11 23:00:00 -0700
---

When I started out with digital electronics, most of my designs ran on 5V from a barrel jack. This was fine for simple
stuff, but I rapidly ran into two problems: 5V isn't high enough for designs that use a lot of power (at least, if you
want to avoid a ton of losses in cables), and barrel jacks aren't super reliable. They rely on just a few points of
contact so it doesn't take much of a bump of the cable to cause a momentary loss of power.

Moving to 12V was the obvious way to solve this (and I did make a few boards that took 12V on a barrel jack), but I
also wanted to move away from barrel jacks. And even 12V starts to become questionable for longer range power
distribution at higher power levels, so a lot of datacenter-type DC buses use even higher voltages, such as 48V.

I also wanted to use a DC bus that was non-isolated (i.e. negative supply rail is at earth ground potential) since a
lot of the projects I have in mind are test equipment or rackmount networking hardware that will have grounded shields
on connectors. It's important to note that I'm using +48V here, rather than the -48V power (positive supply rail at
earth potential) that is commonly used in telco applications.

After a bit of digging, I found that Mean Well makes a power brick (GST280A48-C6P)that puts out ground-referenced +48V
on a locking 6-pin Molex Mini-Fit Jr connector, with up to 280W output. This is enough to run quite a few of my planned
gizmos (most with <25W power budget) from a single DC bus if I made some kind of DC PDU or distribution panel.

There's just one problem: while 48V is great for long range distribution, it's difficult to step directly from 48V to
typical digital core voltages (often <1V for modern devices). You normally need to step it down to an intermediate
voltage, usually something in the neighborhood of 12V, and then go from there to whatever your actual loads require.

Enter the intermediate bus converter.

## Defining the requirements

At a high level, the job of an IBC is pretty simple: take in a high voltage (48V DC in my case) and step it down to a
lower voltage (12V DC). But since this was going to be a system-level power supply, I wanted this to be a bit more than
just a naked buck converter, so I drew up a few initial requirements:

* 48V DC input on a 6-pin Mini-Fit Jr compatible with the previously mentioned Mean Well brick
* 12V DC output on an 8-bit Mini-Fit Jr, maybe PCIe 8-pin compatible (this ultimately didn't happen)
* Remote on/off via a GPIO to support soft power on/off
* Soft start to avoid excessive inrush when driving loads with a lot of input capacitance
* 3.3V DC auxiliary output to power rail/reset supervisors, soft power, and other standby logic
* Temperature, voltage, and current sensors plus an I2C interface for querying them
* Some additional EMI filtering and bulk capacitance

## Version 0.1

The [first iteration](https://github.com/azonenberg/triggercrossbar/commit/30c214f5cfc8a74eedd40133e0658293b26e3e7c) of
the IBC was 87 x 80 mm in size, targeting the OSHPark 2-layer 2oz copper stackup. Back side was almost completely solid
ground with a handful of signal net crossovers, while the front side contained an S-shaped power path with
monitor/control signals around it.

This design set the general stage for all of the subsequent designs, and all future versions have retained electrical
(though not always mechanical) interface compatibility: 5 pin PicoBlade connector containing the 12V enable, I2C, and
3.3V standby rail. The I2C bus contained a temperature sensor at 0x90 and the management microcontroller at 0x42.

The input protection and power path was pretty straightforward: a socketed fuse at the input, common mode choke and
ferrite bead to avoid radiating switching noise out the input, a current shunt and some bulk capacitors, then a
TDK-Lambda i3a series buck module. No explicit reverse voltage or overvoltage protection, although something would
probably blow the input fuse if reversed.

On the output side of the buck module, there's a bunch more capacitace, a current shunt, a ferrite, an an On Semi
NCP455620 controlled-slew load switch.

In parallel with the main power path I put a 3.3V LDO to run the management logic and some voltage dividers to monitor
the 48 and 12V voltage levels, plus a pair of AD8218 current shunt amplifiers to convert the shunt readings to voltages
I could feed to the MCU (a STM32L031).

# Version 0.2

The v0.1 IBC had one major problem: one of the tracks from the output of the buck module to the first capacitor was an
0.125mm track that was supposed to be a marker for an eventual zone fill, but I never added the copper pour! It
functioned fine once I bodged a piece of copper wire across this path.

The v0.1 IBC worked well from a power perspective, but gave very noisy measurements. After a bit of digging I realized
the problem: the ADC bandwidth was high enough that it was picking up switching ripple through the current shunts.
