---
layout: post
title:  "ITCM and veneer adventures"
date:   2024-07-31 23:30:00 -0700
---

In a [previous post](/2024/07/28/Memory-mapping-improvements.html) I discussed how I reached >500 Mbps of iperf3 UDP
performance on an embedded STM32 + FPGA platform.

Unfortunately, the improvements were more brittle than I thought: the loop unrolling seems to have perturbed away the
problem, rather than been the cause of the performance gain. Tiny, unrelated changes like adding a print statement to
boot logic would result in the performance dropping by about 50%.

After some exploration I became convinced that the problem had something to do with the exact layout of the hot path in
the TCP/IP stack and iperf code in memory. Loops aligning (or failing to align) to cache lines, thrashing between
multiple hot functions competing for the same cache line, etc.

## Cache investigations

I spent quite a bit of time reading up on the STM32H735 and Cortex-M7 memory hierarchy, noting such interesting tidbits
as the data cache being 4-way set associative while the instruction cache is only 2-way associative, while both have a
fixed 32-byte cache line (matching the flash write block size of the STM32H735).

I wrote some scripts that parsed objdump output and output a spreadsheet that showed (to the best of my knowledge)
which cache lines each function in the firmware could be allocated to, but didn't find any obvious hot-path conflicts
in the slow firmware that weren't also in the fast version. BSP_MainLoop and APBEthernetInterface::GetRxFrame might
have conflicted, but that's on the RX path and the UDP transmit path would be unaffected (and with 2-way cache
associativity both could be in cache at once).

Ultimately, this ended up being a waste of time. I didn't find any obvious thrashing I could avoid by relocating
specific code.

So I decided to pull out the big hammer and move all of the hot functions over to Instruction Tightly Coupled Memory
(ITCM). Ethernet frame data, stack, and a couple of other critical pieces of state were already in DTCM.

## ITCM setup

If you're not already familiar with the fine points of the Cortex-M7 bus architecture, you might be surprised to learn
that it's actually a sort of Harvard architecture (separate instruction and data buses), although not strict Harvard
since crossovers (execution from D-side bus and data accesses to I-side bus) are permitted with a performance
penalty.

The Cortex-M7 in the STM32H735 has a total of five separate memory buses. Slightly simplifying, these are:
* AXI requester interface for interfacing to flash, bulk SRAM, and external memory buses (FMC and OCTOSPI)
* AHB requester interface for interfacing to most peripherals
* AHB completer interface allowing an external DMA IP to access the ITCM/DTCM SRAMs
* Dual channel 32 bit SRAM interface to DTCM (two separate buses with independent control signals)
* 64 bit SRAM interface to ITCM

The Cortex-M7 architecture allows arbitrarily high latency for TCMs, however the STM32H735 datasheet states that these
are zero-wait-state memories. I'm not clear on if this means no latency *beyond that of an L1 cache miss*, or if it's
truly single cycle access (i.e. TCM access is as fast as an L1 hit) but either way, it's the fastest you can get
deterministically.

The STM32H735 has 64-256 kB of ITCM. No, this isn't a range from model to model within the family, it's dynamically
configurable via the TCM_AXI_SHARED register. Essentially the physical topology is four 64 kB SRAM blocks plus some
muxes allowing three of the blocks to be switched onto the AXI or ITCM buses (the first block is always ITCM).

Actually configuring my firmware to use the ITCM was straightforward. I'm sure there's a different process if you're
using the ST toolchain but I'm using [my stm32-cpp library](https://github.com/azonenberg/stm32-cpp) which did not yet
support ITCM.

First, on the linker script side, I added a memory region for the 64 kB ITCM assuming TCM_AXI_SHARED = 2'b00 (as was
the default, which I hadn't changed).

{% highlight c++ %}
ITCM(RWX):          ORIGIN = 0x00000000, LENGTH = 64K
{% endhighlight %}

Yes, the ITCM is mapped at the all-zeroes address. Meaning a null pointer actually points to the beginning of ITCM, not
an unmapped address!

So when I defined the actual ITCM section in the linker script, I left a blank space at the beginning unused (to make
null pointers point at empty memory rather than executable code). This isn't perfect but filling this space with 0xCC
or similar in the future would enable use of a null pointer to be easily detected.

{% highlight c++ %}
.tcmtext : ALIGN(32)
{
    __itcm_romstart = LOADADDR(.tcmtext);
    __itcm_start = .;

    /* DEBUG: block off first 256 bytes of ITCM since it's mapped at 0 and we want to catch null derefs */
    . += 256;

    *(.tcmtext)
    __itcm_end = .;
} > ITCM AT> FLASH
{% endhighlight %}

This is pretty similar to how you'd specify something like the .data section, in that it lives both in flash and SRAM
and has to take up space in both memories.

The next step was to add an initialization hook to make sure that all of these initialized SRAMs actually got properly
set up at run time. In the case of newlib on ARM, this is done by a global function "hardware_init_hook" that is called
by _start prior to main() or - importantly - __libc_init_array (which calls constructors on global variables before
main() is invoked)

{% highlight c++ %}
extern "C" void hardware_init_hook()
{
    //Copy .data from flash to SRAM (for some reason the default newlib startup won't do this??)
    memcpy(&__data_start, &__data_romstart, &__data_end - &__data_start + 1);

    #ifdef HAVE_ITCM
        //Copy ITCM code from flash to SRAM
        memcpy(&__itcm_start, &__itcm_romstart, &__itcm_end - &__itcm_start + 1);
        asm("dsb");
        asm("isb");
    #endif

    //Initialize the floating point unit
    #ifdef STM32H7
        SCB.CPACR |= ((3UL << 20U)|(3UL << 22U));
    #endif
}
{% endhighlight %}

Nothing particularly out of the ordinary here, the only tricky bit is making sure to do this init in the hook rather
than in main(), which would be too late if any of the TCM functions were called by constructors of global objects.

At this point, the only remaining step was to actually put the hot functions in ITCM. Straightforward enough:
{% highlight c++ %}
#ifdef HAVE_ITCM
__attribute__((section(".tcmtext")))
#endif
void APBEthernetInterface::SendTxFrame(EthernetFrame* frame, bool markFree)
{
{% endhighlight %}

I tested with one or two functions, and after fixing a few typos in the linker script everything was happy and it
worked.

So I started walking my way through the call graph of the hot path in the iperf test, pushing about 4 kB of the most
speed-critical functions into ITCM to see if I could get the consistent high performance I was aiming for. Recompiled
the firmware, flashed it to the board and...

BOOM SEGFAULT

{% highlight plaintext %}
localadmin@fmctest# [2024-07-31T23:20:15.9219] Ready
Hard fault
    HFSR  = 40000000
    MMFAR = 00000000
    BFAR  = 00000000
    CFSR  = 00010000
    UFSR  = 00000001
    DFSR  = 00000002
    MSP   = 2001ff20
    (register dump continues)
{% endhighlight %}

## Investigating the crash

I rebuilt the firmware in debug mode and, of course, the firmware didn't crash with either -Og or -O0. So we're looking
at some kind of heisenbug, great.

I attached gdb to an -O3 binary at the crash and was somewhat confused. It was segfaulting while trying to pop a FIFO.
This was code I had been using for years and, weirder still, the crashing function wasn't even in ITCM (so it shouldn't
have been affected by any of these changes).

{% highlight plaintext %}
HardFault_Handler () at /ceph/fast/home/azonenberg/code/misc-devboards/fpga-stm32-ifaces/firmware/main/vectors.cpp:309
309             while(1)
(gdb) bt
#0  HardFault_Handler () at /ceph/fast/home/azonenberg/code/misc-devboards/fpga-stm32-ifaces/firmware/main/vectors.cpp:309
#1  <signal handler called>
#2  FIFO<EthernetFrame*, 8ul>::Pop (this=0x20006a50 <g_ethIface+24400>) at /ceph/fast/home/azonenberg/code/misc-devboards/fpga-stm32-ifaces/firmware/main/../../../embedded-utils/FIFO.h:106
#3  APBEthernetInterface::GetTxFrame (this=0x20000b00 <g_ethIface>) at /ceph/fast/home/azonenberg/code/misc-devboards/fpga-stm32-ifaces/firmware/main/../../..//staticnet/drivers/apb/APBEthernetInterface.cpp:133
#4  0x08001fc8 in EthernetProtocol::GetTxFrame (this=0x240006f4 <InitIP()::eth>, type=type@entry=ETHERTYPE_ARP, dest=...)
    at /ceph/fast/home/azonenberg/code/misc-devboards/fpga-stm32-ifaces/firmware/main/../../..//staticnet/net/ethernet/EthernetProtocol.cpp:142
#5  0x08000bbe in ARPProtocol::SendQuery (this=0x24000710 <InitIP()::arp>, ip=...) at /ceph/fast/home/azonenberg/code/misc-devboards/fpga-stm32-ifaces/firmware/main/../../..//staticnet/net/arp/ARPProtocol.cpp:53
#6  0x080026e2 in IPv4Protocol::OnAgingTick (this=<optimized out>) at /ceph/fast/home/azonenberg/code/misc-devboards/fpga-stm32-ifaces/firmware/main/../../..//staticnet/net/ipv4/IPv4Protocol.cpp:279
#7  0x08001ffe in EthernetProtocol::OnAgingTick (this=<optimized out>) at /ceph/fast/home/azonenberg/code/misc-devboards/fpga-stm32-ifaces/firmware/main/../../..//staticnet/net/ethernet/EthernetProtocol.cpp:169
#8  0x08007b64 in BSP_MainLoopIteration () at /ceph/fast/home/azonenberg/code/misc-devboards/fpga-stm32-ifaces/firmware/main/mainloop.cpp:111
#9  0x08007902 in BSP_MainLoop () at /ceph/fast/home/azonenberg/code/misc-devboards/fpga-stm32-ifaces/firmware/main/../../..//common-embedded-platform/core/main.cpp:118
#10 0x08007936 in main () at /ceph/fast/home/azonenberg/code/misc-devboards/fpga-stm32-ifaces/firmware/main/../../..//common-embedded-platform/core/main.cpp:87
{% endhighlight %}

The disassembly didn't show anything obviously wrong at a glance.

{% highlight plaintext %}
(gdb) frame 2
#2  FIFO<EthernetFrame*, 8ul>::Pop (this=0x20006a50 <g_ethIface+24400>) at /ceph/fast/home/azonenberg/code/misc-devboards/fpga-stm32-ifaces/firmware/main/../../../embedded-utils/FIFO.h:106
106             objtype Pop()
(gdb) disas
Dump of assembler code for function _ZN20APBEthernetInterface10GetTxFrameEv:
   0x000001fc <+0>:     push    {r3, r4, r5, lr}
   0x000001fe <+2>:     add.w   r4, r0, #20480  @ 0x5000
   0x00000202 <+6>:     ldrb.w  r3, [r4, #3960] @ 0xf78
   0x00000206 <+10>:    cbnz    r3, 0x24c <APBEthernetInterface::GetTxFrame()+80>
=> 0x00000208 <+12>:    blx     0x1000 <__EnterCriticalSection_veneer>
{% endhighlight %}

But wait, what's that "veneer" function?

After some googling I determined this was a thunk added by the linker. Most ARM Thumb jump instructions have a 16-bit
immediate for the destination, and you need a different instruction coding for a far jump. There's a compiler option
you can specify to generate far jumps at the call site, but by default the compiler tries to save a few bytes of code
size by putting the far jump in a thunk and doing a near call to the thunk (thus allowing the far call's instruction
bytes to be reused across many call sites).

This made sense, since GetTxFrame was in ITCM while EnterCriticalSection was an assembly helper in .text. (It might make
sense to move to ITCM since it's small and called frequently, but that's an optimization question and quite orthogonal
to why my firmware is segfaulting.)

So the obvious next step was to look at the veneer.
{% highlight plaintext %}
(gdb) disas __EnterCriticalSection_veneer
Dump of assembler code for function __EnterCriticalSection_veneer:
   0x00001000 <+0>:     bfcsel  0, 0x1a40, 2, ne
   0x00001004 <+4>:     vsub.i32        d16, d12, d0
End of assembler dump.
{% endhighlight %}

I had never heard of bfcsel so I took a look in the ARMv7-M architecture spec... and was rather confused and shocked to
not find it. A bit more research showed that this was an ARMv8-M instruction. So why was I getting one in my ARMv7-M
binary?

I wasn't sure if I was looking at a compiler code generation bug, a gdb/binutils disassembler bug, or something else so
I tried opening the binary in IDA (perks of working in security, always good disassemblers on hand).

![IDA graph view showing an apparently ordinary jump to the veneer](/assets/bad-jump1.png)

Everything looked fine in the outer function, so I moved on to the veneer.

## Bad code generation

![IDA linear view showing opcode 0x04 F0 1F E5 disassembled as "blx.w 0x405a42"](/assets/bad-jump2.png)

This was definitely not right. IDA didn't detect it as a function, and IDA's disassembly didn't match gdb's (neither
made any sense). So some kind of compiler or linker code generation issue.

For comparison, at -O0, I got "ldr.w pc, [pc]" followed by the 32-bit jump destination, which made complete sense. But
for some reason at higher optimization levels we get this garbage instruction.

Looking back at the registers in the crash dump, the CPU agrees with this: the hard fault was actually a usage fault, I
just wasn't getting the usage fault handler called since I never set one up and it got promoted to a hard fault.

UFSR is 0x0000_0001 "undefined instruction executed", so what I thought was a segfault was actually the embedded
equivalent of a SIGILL.

The function I was calling seemed normal enough. All it does is turn off interrupts and return the old CPSR so you can
restore later on. But why was I getting invalid code generation for far calls to it, and not to anything else?

{% highlight assembly %}
.globl EnterCriticalSection
EnterCriticalSection:
    mrs     r0, primask
    cpsid   i
    bx      lr
{% endhighlight %}

After several hours of bashing my head at search results in confusion, I came across [this stackoverflow
post](https://stackoverflow.com/questions/78309423/bad-blx-instruction-generated-when-calling-asm-function-from-c-function-gcc-on).

I still don't understand what's going on, but adding the .type declaration to my function fixed it. This seems like the
kind of thing that should have been caught at link time (the linker knows I'm compiling with -mcpu=cortex-m7 so it
shouldn't generate an instruction that doesn't make sense for that, and if i try to call a symbol that doesn't have a
valid/odd address for thumb code, it should error out rather than generating invalid machine instructions).

Another night gone, now back to other stuff...

Like this post? [Drop me a comment on Mastodon](https://ioc.exchange/@azonenberg/112885517390044964)
