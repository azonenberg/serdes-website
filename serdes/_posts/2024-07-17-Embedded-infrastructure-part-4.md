---
layout: post
title:  "Embedded infrastructure, part 4: Management"
date:   2024-07-17 14:00:00 -0700
---

## The problem

All of these things need network management and/or data transfer capability. The normal solution would be to drop down
a big Cortex-A SoC next to the FPGA running Linux, or use an integrated FPGA+SoC platform like a Zynq.

But embedded Linux comes with a lot of baggage and complexity, so I'm exploring lighter weight alternatives.

## The plan

For the most part, I like to build systems with a clean control/data plane separation with all of the control plane in
a MCU and the data plane living in the FPGA.

On the MCU side I'm very much a fan of simple event loops on a Cortex-M with a few ISRs for handling critical real time
things that I can't offload to the FPGA. No RTOS, no dynamic memory allocation, no unnecessary fluff.

The challenge is that most stuff these days is being optimized for fast development cycles with the lowest paid
developers they can hire, not for this sort of minimalistic design. After a bit of digging I couldn't find a TCP/IP
stack, SSH server, etc. implementation that met my needs.

So I had to start building those from scratch.

## Current state

I have a working TCP/IP stack (IPv4 only at the moment, v6 pending when I have some time) using fully static memory
allocations that runs on a STM32H735, along with a SSH server implementation supporting AES-GCM and (optionally FPGA
accelerated) curve25519 as the only cipher suite. The stack is server only (it cannot initiate outbound connections)
due to it intended use case and omits support for a bunch of features like IP fragmentation that are rarely used.

It's quite lightweight: the trigger crossbar recovery image (SSH server and SFTP firmware updater plus all of the
drivers and utilities required for them to work) clocks in at 82 kB of flash space with -Os, using 114 kB of SRAM
(mostly for socket buffers) plus stack. I could cut the SRAM down even further if I wanted to if I didn't support
several concurrent connections, but to simplify code reuse the recovery image uses the same TCP/IP stack configuration
as the application image. It's not like anything else is going to be using all that RAM during recovery.

The main application firmware is slightly heavier, coming in at 161 kB of flash with -O3 and 124 kB of SRAM - but in
that space I fit all of the hardware drivers and IP stack, SSH management console, serial console, SFTP firmware
updater, SCPI server, DHCP client, SNTP client, and a bunch of other miscellaneous glue.

I have some partial TCP/IP offload capability for high bandwidth data plane stuff in the FPGA, but that's going to need
a bunch more work to get where I want it (it's a lot bulkier than it should be).
