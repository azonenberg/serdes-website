---
layout: post
title:  "Switch project, part 1"
date:   2025-05-07 22:00:00 -0700
---

One of my longest-running projects has been an open hardware Ethernet switch. This has been one of the key driving forces behind many of my other projects, such as ngscopeclient and the high speed probes. It was also the project that got me into high speed digital design.

So I figured it's time to kick off a series with a short writeup of where things are now, how we got there, and what's coming next. If you follow me on Mastodon you've probably seen most of this in bits and pieces but I wanted to collect it all in one place.

## Ancient history: The first switch

The first generation never had a name, it was just called "open-gig-switch" or something in my subversion repository (this was circa 2012 before I was primarily running on git).

[![Dark purple PCB with a mini USB port, quad RJ45, and Spartan-6 FPGA on it](/assets/old-switch-800.jpg)](/assets/old-switch.jpg)

This board pushed a lot of limits for me... my first use of switching power supplies rather than LDOs, my biggest and most complex board to date, and I think maybe even my first use of an RGMII PHY (I had previously used the Microchip ENC424J600 which is a full 10/100 Ethernet MAC + PHY including buffer memory attached to a parallel bus or SPI interface).

And at the time my fanciest piece of test equipment was a 100 MHz Rigol DS1102D. So I had no way to do signal integrity measurements on either the Ethernet differential pairs or even the 250 MT/s RGMII lines.

This one never really got anywhere. I got three of the four PHYs up and running, PHY #4 never worked (I can't remember if it wouldn't link up, wouldn't pass data, or what). I tried resoldering a bunch of stuff but never got it functional.

I tried to bring up a basic switch with only 3 ports, but very quickly realized that 15K LUTs, <1 Mbit of BRAM, and no external CPU would make that very challenging. The XC6SLX25 was not a large FPGA and adding the Xilinx DDR controller and a softcore CPU and three MACs didn't leave much space at all for fabric. Plus I was still a relative novice to FPGA development and wasn't quite at the point of being ready to tackle the project.

So this board ended up a dead-end, but set the stage for what was coming.

## The interim years

The first switch fiasco showed me I needed more skills, better test equipment, better debug tools, and more.

I was still in grad school with little budget for equipment, but over the course of my Ph.D I became a much more experienced RTL engineer. I built a couple of boards with Ethernet that mostly worked.

But since I couldn't afford a better scope and had no way to validate SI on faster stuff, I decided to table the switch project for a bit.

In 2015 I graduated and got a job that paid much more than I was making as a graduate teaching assistant. One of my first purchases was my first "real" oscilloscope, a 350 MHz Teledyne LeCroy WaveSurfer 3034. This was actually fast enough to do SI work on RGMII and protocol-decode 100baseTX, but I dreamed of much faster.

I knew I wanted to put 10GbE on the switch whenever I finally got back to it, and didn't expect to be able to afford a many-GHz scope any time soon, so I started looking at alternative approaches to validate SI at these speeds. This led me down another path of yak shaving in which I started designing a 10 GHz sampling oscilloscope named FREESAMPLE (which never got finished, but I do want to revisit the project one day). This, then, got put on ice when I realized a high speed scope would be useless without an equally high bandwidth probe to feed it with.

You probably already know where this story goes. My open hardware 16 GHz probe (which probably deserves a post or series of its own) is now essentially done and ramping up an initial PVT run, hopefully I can actually make them in quantity and offer to the public in the coming months.

Over the course of shaving these yaks I also bought a house, took a year or two off major projects to refurbish it, got a Sonnet EM solver seat, found a shockingly affordably 16 GHz oscilloscope on eBay (which made FREESAMPLE much less of a priority), rewrote "scopeclient" with OpenGL acceleration as "glscopeclient", then rewrote it again in Vulkan as "ngscopeclient". I also created protocol decodes for 10/100 baseTX, 1000baseX, SGMII, QSGMII, 10Gbase-R, and a few other potentially relevant protocols.

I also started dreaming up a roadmap for a whole family of networking equipment under the randomly generated umbrella name LATENTx, using colors for sub-projects (vaguely inspired by the Lockheed HAVE BLUE). The original roadmap called for LATENTRED to be a gigabit edge switch and LATENTORANGE to be a 10G core switch, but I decided that a prototyping/technology demonstrator platform was called for first. LATENTINFRARED was too long so I went with LATENTPINK as the name.

## LATENTPINK

The original LATENTRED concept had called for 24 edge ports across three 8-port line cards (because multiples of 3 were convenient for using OSHPark for fabrication). RGMII would have been a nightmare to route, since 24 lanes of RGMII need 288 pins (clock, control, 4-bit data bus, times 2 for TX/RX lanes) plus the MDIO, reset, etc. So I started looking at lower pin count options (GMII, needing 576 pins, wasn't even on the table, as no FPGA supported by the free Vivado edition had over 520 GPIOs and anything bigger would be vastly outside my price range anyway).

The obvious "easy" option was SGMII which needs one differential pair each for TX and RX (4 pins). This would only require 96 pins for 24 PHYs, and no parallel bus timing constraints to worry about (just matching P/N of each diff pair). So I started looking at the TI DP83867.

But the project had dragged on long enough that by the time I was ready to make hardware for LATENTPINK it was early 2023 and a new option was on the table. Vitesse, who had previously been on my "naughty list" of companies who will never get a design win from me due to developer-hostile practices like locking datasheets for their most boring products behind NDAs, had been bought by Microsemi in 2015, who had then been bought by Microchip in 2018. As part of this a lot of parts got opened up and I decided they deserved early release for good behavior... which meant that their 12-port QSGMII PHY, the VSC8512, was now on the table. This would allow 24 ports with only six transceiver links (12 diff pairs / 24 pins), a massive reduction from SGMII.

The only problem was, I had never worked with QSGMII before and the VSC8512 needed a fair bit of register configuration to work properly, so I wanted to hedge my bets a bit. This resulted in LATENTPINK, which had one VSC8512 and two DP83867s for a total of 14x 1G edge ports, plus a single 10G SFP+ uplink and a dedicated RGMII management port (using my tried-and-true preferred PHY, the KSZ9031RNX) for the SSH interface.

[![Blue PCB with a row of RJ45 connectors at the front, a SFP+ cage out the back, and several large BGAs in the middle](/assets/latentpink-800.jpg)](/assets/latentpink.jpg)

This was my second 8-layer board (first for a fully personal rather than work-related project), one of my first uses of the Murata MYMGK modules I now use everywhere, and one of my first designs using FPGA transceivers. It was also my first attempt at pairing a STM32H7 with an FPGA (over quad SPI because I hadn't yet learned how much of a disaster the H7 OCTOSPI peripheral was, but it was fine because I was just doing manual register reads/writes and not trying to memory map it).

I got the VSC8512 working fine, the MCU talking to the FPGA (slowly) and used the platform to build out my SSH server and a bunch of other building blocks the full switch would need.

The LATENTPINK board contained an external QDR-II+ SRAM which was used as a packet buffer for a shared-memory based switching fabric. All incoming packets were written to small per-port CDC FIFOs then popped round robin from these FIFOs and written into a region of the QDR serving as a large data FIFO for the port.

On the far side, the forwarding engine would pick a source port round robin then read the QDR FIFO to pop one packet from the port. The output data stream from the QDR would then be written to small per-port exit queues, routed to all of them and with write enables gated according to which port the packet was to be forwarded to.

This design used a home-grown register structure for the control plane and another bus for the data plane, both of which had issues. The control plane bus didn't support any kind of distributed decoding so I had to put all the register logic in one place which required routing decoded SFRs all over the place, and the data plane bus was 32 bits wide for RX and 10G TX, while it was 8-bit for 1G TX. This added some annoying complications to the design and the nonstandard nature meant I couldn't easily reuse FIFOs and other blocks written for other projects.

There were also a couple of PCB bugs, most notably a bad pinout resulting in the upper row of ports only linking up in 10/100 mode until I reworked them (a massive pain given that the swap had to be performed on an inner signal layer, either 3 or 6 depending on the port, in a fairly confined space). I validated the fix on one or two ports but didn't rework the entire board.

Overall I got it to the point that it was functional, it could pass packets, it had port based VLANs and could decode inbound 802.1q tags (but not synthesize outbound tags on trunk ports) before deciding I had proved out the tech stack enough that I was ready to build the real thing.

## What next?

The other major finding from LATENTPINK was that, at least the way I had architected the fabric, the XC7K160T was a bit cramped for my plans. While the 14+1 port design was comfortable, I didn't think it would be easy to fit a 24+2 port switch into it. The next largest 7-series part (the XC7K325T) was not available in the FBG484 package so I'd have to go up to FFG676, but more annoyingly it wasn't supported by the free Vivado edition (requiring a $3K software license) and was hugely more expensive ($2260 vs $435 as of this writing).

Between these two cost adders, I'd be looking at a $5K increase in project cost to jump to the bigger FPGA and build one prototype. My long term goal was 96 ports, so to replace all of my legacy Cisco switches I'd be looking at $3K of software + 4*$2K = $11K *more* to build the entire batch of switches with the 325T vs the 160T. And this would be on top of the already high costs of doing 8-10 layer PCB fab in low volume, the PHYs and RJ45s, custom sheet metal work for the chassis, etc.

By the time I was ready to start thinking about building LATENTRED seriously, though, it was 2024. 7 series was getting pretty long in the tooth and UltraScale and UltraScale+ had been out for a while.

I considered building a switch around the Artix UltraScale+, specifically the largest one - the XCAU25P. It sells for less than the XC7K160T, $380 for the -1 speed grade in FFVB676 as of this writing. It's two process nodes newer (16 nm vs 28) so the fabric is much faster. And at 141K LUTs it's a fair bit larger than the 101K of the 7K160T, although well below the 203K of the 7K325T. Other specs were also mostly in between: 12 transceivers vs 8 or 16, etc. But it was light on block RAM, 10.5 Mb vs 11.7 or 16. And I expected to need a lot of FIFOs in the design, so it would go quickly.

While I did buy a pair of AU25Ps, before I could design a board I was tipped off by a friend to a batch of Kintex UltraScale+'s, specifically the XCKU5P, on AliExpress for a mere $55 each. He had tested one from the seller and they appeared to be legitimate, although likely salvaged/reballed from some scrapped equipment.

This was a major game-changer for the project's direction. The XCKU5P is the largest FPGA supported by the free Vivado license, at 216K LUTs (just larger than the 7K325T), 16 transceivers (matching the 325T), 16.9 Mb of block RAM (slightly larger than the 325T), plus another 18 Mb of UltraRAM, a new kind of large SRAM block optimized for large buffers. The transceivers were also 28 Gbps capable, enabling 25/100G Ethernet rather than only 10/40G. They retail for $2972 in the commercial temperature grade or $3350 in the industrial (which the ones I got were), so I was quite happy with scoring them for less than 2% of list price!

The only problem is, with these new capabilities came scope creep. The 16 transceivers on the KU5P were enough to support twelve lanes of QSGMII (48 baseT ports) plus either a single 40/100G uplink or up to four 10/25G uplinks. And it seemed a shame to not use the full capabilities of such an expensive FPGA falling into my lap for cheap.

## LATENTRED

### Planned hardware architecture

The new concept for LATENTRED is a much more powerful switch than originally planned. It will be a 1U switch with two 24-port line cards (two VSC8512s per line card) and dual 10/25G SFP28 uplinks.

I had initially thought about doing four 12-port cards but found that 6 and 12 port magjacks with AC-isolated center taps (required by the VSC8512's voltage mode MDI drivers) were hard to find, while 8-port ones were available from LINK-PP. The smallest port count evenly divisible by both 8 and 12 is 24, and a 2x12 port line card would just barely fit in my reflow oven.

The overall switch will consist of five, possibly six, PCBs:

* The 48 -> 12V [intermediate bus converter](https://serd.es/2024/10/15/Intermediate-bus-converter.html)
* Power distribution / switching board
* Two 24-port dual VSC8512 line cards
* Switch engine board with XCKU5P, STM32H735, management PHY, serial port, etc.
* Possibly a separate board with the SFP28 uplinks connected to the switch engine by cables, depending on whether the chassis mechanical layout makes this easier than a monolithic design

### Current hardware state

The IBC has been already designed and used in other projects, so that's done. We can forget about it, other than possibly doing that small respin with reduced ripple from the 3.3V buck. But with the current tariff situation I'm not in a hurry (it's a 4L 2oz board made at Multech in China).

The PDU board is done. There's not much to it: 12V in the left 8-pin connector, 12V out the right 8-pin, 12V out the bottom two 4-pins. It also passes the I2C and 3.3V standby rails from the IBC through, while tapping off them to run programmable soft load switching and voltage/current monitoring. Pretty straightforward.

[![Purple PCB with four Molex Mini-Fit Jr power connectors and some passives](/assets/latentred-pdu-800.jpg)](/assets/latentred-pdu.jpg)

The

## Planned switch engine architecture

## Current gateware state
