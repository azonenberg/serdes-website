---
layout: post
title:  "Embedded infrastructure, part 5: FPGA-MCU Bridging"
date:   2024-07-17 21:00:00 -0700
---

## The problem

The main MCU needs some way to talk to the datapath FPGA.

Early versions of the platform just used SPI, but this is slow and a bit awkward since you need to use SFR commands to
read/write all of the registers on the FPGA. Over time various prototypes have shifted a bit in order to simplify this
interoperability.

## The plan

The next step was to convert the internal management bus from an ugly pile of nested case statements to a standard bus
protocol. I decided to go with AMBA APB since this is bridgeable to AXI if needed, but is significantly simpler to
implement and uses much less FPGA resources, while still being more than performant enough for the intended use case.
Ultimately, the goal is a direct memory mapped interface from the MCU to the APB bus that I can use from software just
like an APB peripheral internal to the MCU itself.

The next iteration, as used on the trigger crossbar, was to switch to the STM32H7 OCTOSPI (in quad SPI mode for initial
testing). This ended up being a huge mistake and a waste of a lot of time.

The OCTOSPI allows memory mapping, but a combination of silicon errata and annoying design decisions rendered it
difficult to use: memory mapped reads have a 32-byte prefetch buffer that cannot be disabled, which means any APB read
you issue may result in up to 31 adjacent addresses being read. Subsequent reads to the prefetched address range are
cached in the OCTOSPI peripheral itself (i.e. ignoring any caching properties set in the Cortex-M7 MPU). This means
that reads with side effects will potentially trigger randomly when adjacent registers are read, and polling loops are
very awkward to implement since the slightest misstep can lead to you infinite-looping with the poll hitting in the
cache and not actually querying the peripheral.

On top of these wrinkles, it doesn't support backpressure so there's no way to stall if a transaction takes more time
to execute. You need to have a fixed number of dummy clocks on the QSPI and the APB has a hard realtime constraint
based on that turnaround time.

So I spun a test board to try out the other memory mapped external bus, the FMC (flexible memory controller).

## Current state

Other than needing a lot of pins (~27 depending on exactly which modes are active and how much address space you want
to map), the FMC is everything I could hope for: it supports device-mode memory mapping (completely uncached strongly
ordered access), there's no caching in the peripheral, and it Just Works(tm). It supports backpressure and you can even
set it to generate a free-running clock that you can lock a PLL to, etc. The only APB feature of interest that it
doesn't natively support is the ability to propagate a bus error back from APB PSLVERR; if necessary I can hook
PSLVERR to a GPIO that triggers an interrupt that I treat as equivalent to a bus error.

There's a small silicon errata causing two dummy clocks to be added to the end of a read burst, but that's simple
enough to ignore on an FPGA although it does cause a small performance degradation. It can clock up to 275 MHz (max APB
frequency of the STM32H735) although right now I'm testing at 125 on a slow Spartan-7; actual deployments with an
UltraScale+ FPGA will have no problem running at the 250 MHz I intend to use.

I'll do a more detailed writeup on the setup once I've got it fully packaged up and more extensively tested, but at
this point it's looking like this will be my interface of choice moving forward. I have it working well,
bidirectionally bridging from AXI on the STM32 to APB in the FPGA.

It would be nice if the FMC supported arbitrary-sized burst accesses (or even just 64-bit bursts) to reduce overhead
for bulk data transfer since the AXI on the STM32 side is 32 bits wide, but so far in my tests all attempts to do
64-bit AXI transfers have resulted in two consecutive 32-bit FMC accesses. I'll need to play around a bit and see what
kind of opcodes the compiler actually generated.
