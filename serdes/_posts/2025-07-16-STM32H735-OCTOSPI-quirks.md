---
layout: post
title:  "STM32H735 OCTOSPI quirks"
date:   2025-07-16 07:30:00 -0700
---

If you've been following this blog for a while, you probably know that I do almost all of my high-end embedded work by pairing a STM32H735 with a Xilinx FPGA using the external parallel memory controller (FMC) in PSRAM mode. It works great, you can get 128 MB of off-chip address space bridged pretty directly to the internal AXI bus and if you set the MPU right you can make it behave as strongly ordered device memory with no caching etc, just straightforward MMIO.

There's just one problem: the FMC uses a *lot* of pins. A full FMC interface uses a whopping 34:
* Clock
* Chip select
* 4 control signals (NWAIT, NOE, NWE, NL/NADV)
* 16-bit muxed address/data bus
* 2 byte mask strobes
* 10 additional address bits

You can cut this down as far as 24 pins if you don't need all of the address bits (the minimal configuration can address 2^16 16-bit words or 128 kB of register space) but that's still quite a lot of signals to route. If you're on a low layer count board, or need as many GPIOs as possible on your FPGA or MCU, or are trying to cut costs and simplify routing by using the 68-QFN package for the MCU rather than one of the BGA offerings, the FMC isn't an option.

Or maybe you're just me, stuck writing firmware for an expensive board that already exists and didn't pin out the FMC because I hadn't learned about it yet.

Anyway, no pretty pictures in this one either. Just a quick infodump of things I've learned while dealing with this monstrosity.

## Enter the OCTOSPI

Luckily (at first glance) the STM32H735 provides an alternative: the OCTOSPI. This is a peripheral designed to handle serial and small-width parallel memories (abbreviated xSPI here for conciseness) such as commodity SPI/QSPI flash, HyperRAM, QSPI NAND, etc. It offers the ability to run in indirect mode (write address and data to SFRs to initiate a transaction) or memory mapped mode (the external memory is bridged directly to AXI). You only need a handful of pins (six in QSPI mode), and it can clock reasonably fast (up to 140 MHz at 3.3V Vdd).

And it's one of the most cursed microcontroller peripherals I've worked with in a long time. If you try to use it for external RAM you're in for a world of hurt (see [this comment thread](https://news.ycombinator.com/item?id=41073005) for some specifics on some of the failure cases - yes, this is some of the best information out there and that says a lot).

But if you're willing to jump through some hoops, you can make it work reliably for my use case (interfacing to an FPGA, where you control both sides of the link, and are only accessing 32-bit SFRs). It took me a long time to learn all of the undocumented quirks of the peripheral and I'm writing this to share what I've learned.

## Prefetching and caching

It's pretty clear that MMIO was not an intended use case for the OCTOSPI, it's designed for XIP from external flash or interfacing with PSRAM for the most part. (This is ironic, given how buggy it is for this use case).

First off, there is a 32-byte prefetch buffer and cache *inside the peripheral*. It's important to understand that this feature is always enabled (when in memory mapped mode). There's no documented register to turn it off (I even opened a support case to inquire about undocumented chicken bits and was told none existed), and it's independent of and lower level than the CPU cache. So even if you set the MPU to have the OCTOSPI address range as strongly ordered/device memory with no caching, you are still running through the cache inside the peripheral.

Any time you issue a read transaction using the OCTOSPI in memory mapped mode, it will initiate a read burst on the xSPI bus, then read up to 32 bytes (regardless of alignment) until the prefetch buffer is full or the burst is terminated by a subsequent read to a different address.

If you then issue an AXI read to any address in the prefetched range, the read will hit in the cache immediately and no xSPI transaction will be issued.

The upshot of this for FPGA interfacing is:
* Registers must not have side effects (popping a FIFO, clearing a status flag, etc). on read. Any read of any address up to 0x20 bytes before a given register may result in a spurious read transaction to that register as part of the prefetch operation. Instead, an explicit write to a control register to perform this operation must be performed by the application.
* Polling of a status register (to wait for data to arrive, busy bit to clear, etc) is impossible, since multiple consecutive reads to the same register will always result in a cache hit. The easiest workaround is to map every status register you might want to poll at two different addresses 0x20 apart and poll them in a ping-pong fashion, forcing a cache miss and new xPSI transaction to the FPGA each time.

## Read backpressure and bus hangs

The OCTOSPI supports a DQS/RWDS pin (mostly intended for HyperRAM). If you hook this pin up and enable it for reads, you can use it to provide a backpressure mechanism that stalls the memory controller if your FPGA isn't ready to service a read yet (e.g. because an on device bus transaction hasn't finished).

There are a few dragons here: The HyperBus protocol normally only supports single or double latency for reads, but the OCTOSPI allows arbitrarily long backpressure. This is good in that you can stall for a variable number of clocks if your FPGA-side bus isn't ready, but bad in that a bug in the FPGA bitstream (or worse yet, accidentally enabling DQS when the pin isn't actually hooked up) can lead to an unbounded stall on the AXI bus.

This results in a hard lockup of the MCU, so hard that you can't even access it via JTAG/SWD (since the MEM-AP is accessing CoreSight registers on the same AMBA fabric) to troubleshoot or load a fixed firmware. As a general guideline, any time you are developing firmware that has an external bus interface, you should provide an exit path. Some options I've used over the years:

* GPIO you can jumper to a state that results in an infinite loop or entry to bootloader mode prior to bringing up the external memory
* Timeout where external memory isn't initialized for a few hundred ms after reset
* Don't initialize the external memory bus automatically on boot at all, wait for a CLI command or something

## Write clobbering and coalescing

The previously mentioned DQS/RWDS pin is also used (in HyperRAM) as a byte write enable strobe.

Due to errata 2.7.6, the WCCR.DQSE bit *must* be set in memory mapped mode when performing writes - otherwise you get an AXI bus fault any time you try to write to the memory. It's unclear how much of the behavior described in this section is a consequence of having DQSE set, and how much is intendeded behavior / would still apply to future devices that have this errata fixed.

First off, the OCTOSPI will always issue a minimum of an 8-byte burst when doing writes (i.e. a full 64-bit AXI transaction). If you didn't hook up the RWDS pin (as is the case for my board, which was designed before I became aware of the errata) there's no way to know that the second write isn't intentional (I'm not sure if RWDS is properly set or not, the errata is confusing and I don't have enough LA captures of other boards to be sure). This means that if you write to a 32-bit SFR, the adjacent one is going to get corrupted.

Second, worse yet, the OCTOSPI does burst combining: if you write to two locations up to 0x20 apart in quick succession, rather than issuing two separate 32- or 64-bit write bursts, it will issue a single long write burst (and at least in theory, use RWDS to mask off the writes to the in-between location). The end result is that all registers in between the two you intended to write get corrupted.

## Conclusions

In short, if you plan to use the OCTOSPI, the following rules will avoid most of the pain:
* Don't try to use it as RAM. Only use it for FPGA interfacing (or possibly read-only memory mapped flash? I haven't validated this use case though)
* All registers must be 32 bits in size, accessed only as full 32 bit words (no byte/halfword access allowed) and aligned to 0x20 byte boundaries with padding in between them. The one exception is a large scratchpad buffer (e.g. Ethernet frame transmit buffer) that you will be reading/writing as a block of 32-bit words in strict linear order, in which case it's OK to just map the whole thing as a consecutive region. An explicit write-length register is required since garbage/padding is appended to the end of a write burst, so you can't rely on writing exactly the number of bytes you intended to send.
* Registers must not have side effects on read.
* If you need to read the most up-to-date value from a register, make sure the most recent previous read was from at least 0x20 bytes away from it to ensure a cache miss. If you don't know what the last read was, issue an explicit dontcare read from a different register before reading the register of interest.
* Any register you plan to poll should be mapped at two different locations in the address space, separated by at least 0x20, and polled in a ping-pong fashion
* Enable the DQS pin as an output, even if you didn't hook it up to the FPGA.
* If you aren't using DQS as an input on the MCU, there is a hard realtime constraint for your FPGA-side bus due to the nature of the xSPI protocol: you must service a read in time for the data to go out on the bus. You can adjust the xSPI clock rate and latency values to provide as much time as needed for this, but at the cost of a fixed overhead on every bus transaction (so variable latency isn't possible).
* If you do hook up DQS as an input, provide a way to recover from an AXI bus hang during development (you should probably provide this even if you're not intentionally using DQS input, just in case you accidentally turn it on with a bad register config).

I think I covered anything, if I missed something please let me know.

Like this post? [Drop me a comment on Mastodon](https://ioc.exchange/@azonenberg/114863562354865441)
