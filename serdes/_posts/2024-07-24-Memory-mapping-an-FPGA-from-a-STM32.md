---
layout: post
title:  "Memory mapping an FPGA from an STM32"
date:   2024-07-24 21:00:00 -0700
---

I teased at this a bit in my previous posts and finally have a setup I'm happy with, so I thought I'd do a more
in-depth writeup.

To recap, the planned architecture for most of my future large-scale embedded projects is a fairly large (AMD Xilinx
Kintex-7 or Artix / Kintex UltraScale+) FPGA for the high speed data plane paired with a STM32H735 for the control
plane with a memory mapped interface between them.

## Why a two-chip solution?

This is somewhat reminiscent of SoC FPGAs like Xilinx's Zynq / Versal platforms, but with a few important differences
that make it suit my needs and preferences better:

* Using a MCU-class Cortex-M CPU instead of an applications processor is simpler to program in a bare-metal no-OS
or minimal RTOS environment
* The large (564 kB) on chip SRAM and 1 MB on chip flash eliminates the need for time-consuming DDR SDRAM layout
* The disaggregated pinout (two smaller BGAs rather than one larger one) is simpler to fan out on less PCB layers, and
allows placing the FPGA and MCU with some distance between them if this is more convenient for layout reasons
* Decentralizing allows the FPGA and MCU to enforce security boundaries between each other. For example, the FPGA can
refuse to accept a bitstream from the MCU that isn't signed by a key stored in the FPGA, and on-MCU memory and
peripherals can't be touched by the FPGA.
* I can mix and match MCUs and FPGAs to achieve the mix of features, IO, BOM cost, etc. that fits my application
* The STM32 has hardware AES and random number generator IPs while Zynq (shockingly) does not

## The memory interface

After several false starts using quad SPI, I've settled on using the Flexible Memory Controller (FMC) as the preferred
MCU-side bridge between the AXI on the STM32 and the FPGA's internal interconnect. This is a highly configurable module
which can be used to interface to old school (PC133 etc) SDRAM, asynchronous or synchronous SRAM/PSRAM, parallel
NOR/NAND flash, etc.

Most importantly, unlike the OCTOSPI peripheral on the STM32H735, there is no hardware caching or prefetch in the FMC
IP itself - only the normal L1 I/D caches provided by the Cortex-M7. And there's not even any need to mess with these,
since the first FMC bank has a mapping at 0xc000_0000 which is strongly configured as ordered, uncached, device memory
in the MPU right out of the box - no need to mess around with MPU registers to turn off the cache for this range.

The FMC operating mode most amenable to FPGA bridging is synchronous PSRAM since this provides a clock (which can be
made free-running between memory activity burst, allowing you to run internal FPGA logic, PLLs, etc. off it) and
supports a hardware wait signal, allowing the FPGA to stall the bus in case pipeline delays or a slow peripheral
require more latency than the fixed number of wait states provided in the initial FMC register configuration.

Hooking up 26 LVCMOS33 pins including clock, 16 bit multiplexed address/data, control signals, byte write enables, one
chip select, and three high address bits occupies about half of a 7 series HR or UltraScale+ HD I/O bank. This
configuration gives me 1 MB of address space (2^(16+3) = 512K 16-bit words addressable) on the FPGA and I can break out
a few more address pins if I need more space for a more complex FPGA design.

## Hardware design

I didn't have any boards with a suitable combination of MCU, FPGA, and interconnect routing so I threw together a quick
test board in KiCAD. It's a six layer design on Shengyi S1000-2M (cost optimized Asian FR-4 class material) since
there's nothing particularly fast on the board and I wanted it to be cheap.

(board photo here)

The board is intended to pair with my second generation 48 -> 12V [intermediate bus
converter](https://github.com/azonenberg/common-ibc) and also be used for bringup/validation testing of it, so it
includes the PicoBlade control and Mini-Fit Jr 12V input connectors for that. I have the new IBC boards but haven't had
time to populate any yet, so for now I'm powering it off a first-iteration reworked IBC prototype that I had lying
around.

It contains a STM32L431 in QFN-48 as supervisor / rail sequencer (so I can validate that and develop firmware before
using it in a more expensive, complex design), a STM32H735 in 201-BGA as the main processor, and a Xilinx XC7S25
Spartan-7 FPGA in FTGB196 for the other side of the bridge. I could have got away with less FPGA but wanted this board
to be a more general FPGA+MCU dev board, and also needed sufficient logic capacity and RAM for logic analyzer cores
during bringup, so decided against using the XC7S6 or 15 that I had on the shelf.

The FPGA and MCU are wired together by several interfaces: the FMC discussed above, an OCTOSPI channel, and 10/100
RMII. The OCTOSPI and RMII are not used in the current firmware due to the caching issues discussed in my previous
posts and the fact that the FMC is significantly faster than the RMII interface (more on that later).

The second OCTOSPI channel on the MCU is connected to a quad SPI flash that is currently unused, but I want to play
with in the future. I think this will work fine; the OCTOSPI is actually designed for interfacing with external flash
and most of the quirks I've encountered were the result of trying to shoehorn it into something it was not meant to do.

In addition to the interfaces to the MCU, the FPGA has an RGMII connection to a KSZ9031RNX gigabit Ethernet PHY, a PMOD
for GPIO expansion, and four LEDs for status indications.

The MCU also has a PMOD of its own, another four LEDs, and a 3.3V UART broken out to pin headers for debug console.

## Integrated platform

The STM32H735 is a very complex chip (the [reference
manual](https://www.st.com/resource/en/reference_manual/rm0468-stm32h723733-stm32h725735-and-stm32h730-value-line-advanced-armbased-32bit-mcus-stmicroelectronics.pdf)
weighs in at 3357 pages) so we'll only show the parts relevant to the discussion in figures here.

From the MCU's perspective, the FPGA shows up as a 64 MB region (of which only 1 MB is wired up on this board) of APB
SFR address space mapped starting at 0xc000_0000. In my [linker
script](https://github.com/azonenberg/misc-devboards/blob/master/fpga-stm32-ifaces/firmware/main/firmware.ld) these
regions are referred to as FMC_APB1 and FMC_APB2 to avoid ambiguity with the on-MCU APB1 and APB2 bus segments located
in the 0x4000_0000 peripheral address range.

64-bit accesses are not currently supported since the FPGA-side bus is 32 bits and I haven't implemented logic to break
up a 64-bit burst into two 32-bit transactions. 32-bit read and write accesses are fully supported including wait state
propagation; 16 and 8 bit accesses are mostly implemented but thorough testing has been low priority since most
of my peripherals have native 32-bit registers anyway.

(image here)

The FPGA design (implemented in SystemVerilog) contains:

* A tri speed 10/100/1000 [RGMII
MAC](https://github.com/azonenberg/antikernel-ipcores/blob/master/interface/ethernet/RGMIIMACWrapper.sv?ts=4) paired
with memory mapped [RX
FIFO](https://github.com/azonenberg/antikernel-ipcores/blob/master/interface/ethernet/APB_EthernetRxBuffer_x32.sv) and
[TX
FIFO](https://github.com/azonenberg/antikernel-ipcores/blob/master/interface/ethernet/APB_EthernetTxBuffer_x32_1G.sv)
* An [MDIO
controller](https://github.com/azonenberg/antikernel-ipcores/blob/master/interface/ethernet/APB_MDIO.sv?ts=4)
* A 32-bit
[GPIO port](https://github.com/azonenberg/antikernel-ipcores/blob/master/interface/gpio/APB_GPIO.sv?ts=4) connected to
the PHY reset pin, some other miscellaneous control/status signals around the board and internal to the FPGA, and the
FPGA PMOD pins
* A few support blocks for things like querying the FPGA device ID, temperature, and other system health sensors

## FPGA-side AMBA implementation

I shied away from using AMBA interconnects in my FPGA design for a long time because of Xilinx's choices of using AXI
(large and heavy weight) for everything, and having individual ports for every control signal (whyyyy?). But Vivado now
has good support for SystemVerilog interfaces (when I last looked at this circa 2017, while interfaces were supported
it couldn't handle arrays of them).

Rather than using AXI for everything, I've decided to standardize on 32-bit APB as my internal control-plane
interconnect. It's much smaller and simpler, and for poking config registers it's more than fast enough. And, as you'll
see later on, you can actually push quite a bit of data over it if necessary.

The [top
level](https://github.com/azonenberg/misc-devboards/blob/master/fpga-stm32-ifaces/rtl/iface-test/iface-test.srcs/sources_1/new/top.sv?ts=4)
of the design is mostly IO declarations and the APB interconnect.

### FMC bridge

The FMC bridge is a bidirectional converter between the STM32 FMC bus to AMBA APB. You can go look at [the
source](https://github.com/azonenberg/antikernel-ipcores/blob/master/amba/bridging/FMC_APBBridge.sv?ts=4) if you want
to see all the gory details, but here's how it's instantiated.

The bridge contains an internal PLL (as of this writing only 7 series is supported but UltraScale+ will be added soon)
locked to the FMC clock (which must be free running) and generating two equal-frequency output clocks. The phase of
these clocks can be adjusted as needed to improve setup/hold margin depending on on IO timing requirements for your
specific FPGA, board trace delays, etc.

The first clock phase is used for capturing inbound FMC control/write data signals and driving the APB PCLK out to
internal loads within the FPGA, while the second clock is used for launching read data back to the MCU. At higher clock
speeds it may be necessary to move the launch clock back relative to the capture clock in order to buy a bit more
timing margin for the system-synchronous bus. (If anyone at ST is listening, could you maybe add some kind of DQS or
other source-synchronous read capture clock to the FMC in your next gen parts?)

This bridge converts inbound FMC transactions directly to APB read/write transactions, setting the APB PSTRB signal as
needed to match the byte write enables on the FMC. APB latency is properly propagated back to the NWAIT signal on the
FMC, so peripherals can take arbitrarily long to service requests (although this will stall the AXI bus on the STM32,
so beware).

As of now, the PSLVERR signal is not used for anything, but in the future I plan to break it out to a latching
interrupt line of some sort that will trigger a "you done segfaulted" ISR on the MCU to handle the error.

{% highlight verilog %}
APB #(.DATA_WIDTH(32), .ADDR_WIDTH(20), .USER_WIDTH(0)) fmc_apb();

FMC_APBBridge #(
	.CLOCK_PERIOD(7.27),	//137.5 MHz
	.VCO_MULT(8),			//1.1 GHz VCO
	.CAPTURE_CLOCK_PHASE(-30),
	.LAUNCH_CLOCK_PHASE(-30)
) fmcbridge(
	.apb(fmc_apb),

	.clk_mgmt(clk_125mhz),

	.fmc_clk(fmc_clk),
	.fmc_nwait(fmc_nwait),
	.fmc_noe(fmc_noe),
	.fmc_ad(fmc_ad),
	.fmc_nwe(fmc_nwe),
	.fmc_nbl(fmc_nbl),
	.fmc_nl_nadv(fmc_nl_nadv),
	.fmc_a_hi(fmc_a_hi),
	.fmc_cs_n(fmc_ne1)
);
{% endhighlight %}

### APB bridges

My [APB bridge](https://github.com/azonenberg/antikernel-ipcores/blob/master/amba/apb/APBBridge.sv?ts=4) module takes a
single APB requester port and bridges it out to arbitrarily many completers, each mapped at consecutive, equally sized
regions of address space configured as an array of SystemVerilog interfaces. No fancy GUI address space editors, no
automatic code generation, just a parameterizable module.

Most nontrivial designs will include a mix of peripherals with simple, tiny register maps (just a few control bits) and
larger, more complex ones with memory mapped buffers etc. My architecture implements this by using a tree of bridges;
the test system has a root bridge with two 64 kB bus segments. One of these is then subdivided into 1 kB segments for
general peripherals while the other is subdivided into 4 kB segments for the Ethernet FIFOs.

The bridge is completely combinatorial to provide maximum flexibility for timing-latency tradeoffs; it is expected that
real-world designs will add [register
slices](https://github.com/azonenberg/antikernel-ipcores/blob/master/amba/apb/APBRegisterSlice.sv?ts=4) throughout the
design as required to make timing at the desired PCLK frequency.

{% highlight verilog %}
//Two 16-bit bus segments at 0xc000_0000 (APB1) and c001_0000 (APB2)
APB #(.DATA_WIDTH(32), .ADDR_WIDTH(16), .USER_WIDTH(0)) rootAPB[1:0]();

//Root bridge
APBBridge #(
	.BASE_ADDR(32'h0000_0000),	//MSBs are not sent over FMC so we set to zero on our side
	.BLOCK_SIZE(32'h1_0000),
	.NUM_PORTS(2)
) root_bridge (
	.upstream(fmc_apb_pipe),
	.downstream(rootAPB)
);

//Pipeline stages at top side of each root in case we need to improve timing
APB #(.DATA_WIDTH(32), .ADDR_WIDTH(16), .USER_WIDTH(0)) apb1_root();
APBRegisterSlice #(.DOWN_REG(0), .UP_REG(0)) regslice_apb1_root(
	.upstream(rootAPB[0]),
	.downstream(apb1_root));

APB #(.DATA_WIDTH(32), .ADDR_WIDTH(16), .USER_WIDTH(0)) apb2_root();
APBRegisterSlice #(.DOWN_REG(0), .UP_REG(0)) regslice_apb2_root(
	.upstream(rootAPB[1]),
	.downstream(apb2_root));
{% endhighlight %}

## Performance

In order to check how fast the interface actually is, I wrote a minimalistic iperf3 compatible server application as a
benchmark. Not that I actually expect to be trying to firehose packets as fast as I can (I didn't implement rate
limiting) from a STM32 hanging off an FPGA, but it's a decent stress test of the interconnect bandwidth.

I chose reverse UDP mode (STM32 sending, PC receiving) for the benchmark to minimize the amount of CPU used on the
benchmark with the goal of primarily stressing the bus - in other words, this should not be interpreted as a realistic
performance figure that can be achieved by actual application code doing nontrivial things, merely a figure of merit
for comparison to future implementations.

The current APBEthernetInterface driver doesn't use any DMA, just a busy loop that effectively memcpy's the data over
(with a few slight tweaks to ensure alignment etc). Given that all of the memory accesses are made by the CPU, I put
the packet buffers and all of the internal data structures used by the TCP/IP stack in DTCM to maximize performance.

With the FMC clocked at 125 MHz and both PLL clock phases set to -30 degrees (after BUFG insertion delay), my current
test firmware can sustain 284 Mbps over a ten-second test.

(image here)

(TODO can we make it faster)

## Conclusions

Overall this was surprisingly painless. The interface just works, with almost no fuss. Pushing to higher clock rates
(past 125 MHz) is likely to be a bit challenging due to the system-synchronous nature of the bus. I played around a bit
with dynamic PLL reconfiguration and some ideas for link trainining of sorts, but honestly I don't think it's
necessary. It's more than fast enough for my use case.

