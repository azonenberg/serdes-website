---
layout: post
title:  "Switch project, part 2 - Line Card"
date:   2025-06-21 01:00:00 -0700
---

This is part 2 of my ongoing series about LATENTRED, my project to create an open source 1U managed Ethernet switch
from scratch.

Today, we're going to be talking about the 24-port QSGMII line card. Two of these will provide all of the front panel connectivity for the edge-facing side of the switch (i.e. everything but uplink and management ports).

## Overview

Here's the line card before heatsink installation. This is "upside down" in the canonical CAD layout, but provides a better view of the layout without the RJ45s blocking the view.

[![Blue PCB with three 8-port RJ45s and two large BGA PHYs,](/assets/linecard-800.jpg)](/assets/linecard.jpg)

The overall layout of the board is mostly symmetric, but with some variations because of the PHYs not being mirror images of each other.

At the north side we have the three 8-port RJ45s. These are Link-PP LPJT476156AENL and among the most expensive parts on the board at $33 each (plus tariff for any I need beyond the initial ten samples I got for prototyping), second only to the $40.11 PHYs.

Speaking of PHYs, the two large BGAs are Microchip (formerly Microsemi, formerly Vitesse - yay semiconductor industry M&A) VSC8512 12-port QSGMII PHYs. Each PHY is responsible for half of the front panel ports on the line card (one quarter of the whole switch).

These are older 65nm parts, but as of when I last checked they were the only 12-port QSGMII PHY available in low volume with no NDAs etc, so I'm stuck using them. One of their quirks is that the low speed digital GPIOs (MDIO, reset, JTAG, etc) are LVCMOS25 rather than the LVCMOS33 or LVCMOS18 in more common use in modern parts. This doesn't change a whole lot on the line card board, but does cause some annoyances on the switch engine board which we'll discuss in a future post. Each PHY has a probably-unnecessary JTAG connector, using the 2x7 Xilinx pinout since I have a lot of those dongles, in case I want to do boundary scan test. So far I haven't needed this.

On the far east/west sides of the board next to the JTAG connectors are a series of power rail test points (more on that in a bit) and filtering components for the various isolated analog domains on each PHY. This area also contains an AT30TS74 I2C temperature sensor to measure board temperature in the general area of the PHY.

Directly to the south of each PHY is a Samtec ARF6 series connector. This carries eight 100Ω differential pairs, although only six are used here (three TX and three RX lanes). Using this "flyover" architecture allows the PCB to be made of an inexpensive material (Shengyi S1000-2M, relatively standard high-Tg FR-4) without the 5 Gbps signals suffering significant losses from long distance routing on a high-Df material. These links are AC coupled on the line card for both TX and RX. Slightly to the west of each is a pair of SMPM connectors which can be configured to output a recovered clock if I ever need this for validation/test purposes.

In the far southwest corner are power and reset buttons for debug. These drive GPIOs on the supervisor MCU and aren't intended to be used in the final system, which will allow the line card to be remotely managed over an I2C or SPI bus from the FPGA or main processor.

Just west of the centerline along the south edge of the board is a ten-pin Molex PicoBlade connector which carries all of the low-speed management traffic: I2C to the MCU and sensors, SPI if I want more bandwidth to the MCU, and the MDIO bus for configuring the PHYs.

Down the centerline, we have the power supply - four Murata MYMGK00504ERSR 4A DC-DC modules. These are all fed by a common 12V input on the 4-pin Molex Mini-Fit Jr connector at the south center edge of the board, and each drives one of the board's four core power rails: 3.3V, 2.5V, 1.0V digital, and 1.0V analog. This area also has another AT30TS74 to monitor PSU temperature, and an INA230 shunt monitor on each rail to monitor current and voltage.

Finally, sandwiched between the PSU and the PHY to its west, is the supervisor. This is a STM32L431 that controls enables and sequencing for all of the power rails, reset and power-down control for the PHYs, and monitors for fault conditions such as a power rail dropping out of regulation or a temperature limit being hit.

## Power

I did initial power supply validation on the line card sitting at idle with 23 of 24 ports linked up, 22 through a set of 11 loopback cables and the 23rd cabled to the sandbox VLAN on my lab network. There's no traffic flowing other than normal ambient network broadcasts on port 0 because I haven't finished the gateware enough to actually be passing packets, but this is the best I can do at the moment.

Voltage and ripple measurements were taken with a Teledyne LeCroy WaveRunner 8404M-MS oscilloscope and RP4030 active power rail probe. Current measurements used the shunt resistors and INA230s integrated into the on-board PSU.

(probe pic todo)

### 12V

TODO

### 3.3V

This rail feeds the supervisor, I2C sensors, and the high side of the port status indicator LEDs. It's always on, since the line card doesn't have 3V3_SB routed to it and I didn't see the point in adding a fifth regulator since it has no sequencing requirements anyway.

linecard-3v3 TODO

Voltage is 11 mV, or 0.3%, below nominal. Considering I used 1% resistors, I'm happy with that.

Current consumption is around 174 mA at 3.289V, for a total of 572 mW.

I think a sizeable amount of this is in the port indicator LEDs (which are rated for 20 mA each although I'm driving them slightly less hard than that). I'm not sure why RJ45s always seem to be using the old yellow-green AlGaP LEDs rather than the modern emerald-green InGaN ones, I could probably cut almost a watt off my system power budget if I were able to switch, but it was hard enoguh to find high-density jacks with integrated magnetics and AC coupled center taps for voltage mode PHYs.

Ripple is very good, 15.8 mV P-P and 2.52 mV RMS. It's dominated by 300 kHz switching ripple with high frequency noise at 125 MHz and harmonics thereof (obviously coming from the PHYs, maybe the LED drive circuitry or something). There's also a 95 MHz spectral line I can't readily explain. I thought it was the supervisor MCU core clock at first, but that's running at 80 MHz so as soon as I dropped a cursor on the peak I discarded that hypothesis... but it's at like -75 dBm so I don't particularly care. It's not remotely strong enough to be a problem.

### 2.5V

This is by far the biggest power hog on the board, driving both the LVCMOS GPIO pads on the PHYs (for MDIO and LED drive) directly, and (through a pi filter network) the analog PAM-5 drivers for the twisted pair.

linecard-2v5 TODO

Voltage is 21 mV, or 0.84%, below nominal, Also well within acceptable limits.

Ripple is excellent, 6.9 mV p-p and 788 μV RMS.

TODO: talk about analog rails

## QSGMII

## Ethernet MDI

## Thermals
