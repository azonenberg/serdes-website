---
layout: post
title:  "Memory mapping an FPGA from an STM32"
date:   2024-07-24 21:00:00 -0700
---

I teased at this a bit in my previous posts and finally have a setup I'm happy with, so I thought I'd do a more
in-depth writeup.

To recap, the planned architecture for most of my future large-scale embedded projects is a fairly large (Kintex-7 or
Artix / Kintex UltraScale+) FPGA for the high speed data plane paired with a STM32H735 for the control plane with a
memory mapped interface between them.

This is somewhat reminiscent of SoC FPGAs like Xilinx's Zynq platform, but with a few important differences that make
it suit my needs and preferences better:

* Using a MCU-class Cortex-M CPU instead of an applications processor is simpler to program in a bare-metal no-OS
or minimal RTOS environment
* The large (564 kB) on chip SRAM and 1 MB on chip flash eliminates the need for time-consuming DDR SDRAM layout
* The disaggregated pinout (two smaller BGAs rather than one larger one) is simpler to fan out on less PCB layers, and
allows placing the FPGA and MCU with some distance between them if this is more convenient for layout reasons
* Decentralizing allows the FPGA and MCU to enforce security boundaries between each other. For example, the FPGA can
refuse to accept a bitstream from the MCU that isn't signed by a key stored in the FPGA, and on-MCU memory and
peripherals can't be touched by the FPGA.

## Block diagram
