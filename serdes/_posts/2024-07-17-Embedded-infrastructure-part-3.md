---
layout: post
title:  "Embedded infrastructure, part 3: The supervisor"
date:   2024-07-17 12:00:00 -0700
---

## The problem

Most of my ongoing projects need a lot of power rails. My trigger crossbar, for example, has ten independently
controllable power domains (not counting the 48V input, analog rails which are isolated by pi filters but not switched
or regulated separately, and the dynamically scalable Vcore on the processor which is controlled by firmware on the
processor itself).

Some of these rails have sequencing requirements relative to each other, plus on hand assembled prototypes there's a
very real risk of a solder defect shorting one, so just letting them all coming up uncontrolled isn't an option.

The high-volume-product solution to this sort of thing is a dedicated supervisor IC and/or chaining PGOOD and ENABLE
signals on regulators with hard wired connections.

But in a prototype or low-volume device, this is problematic since unexpected sequencing requirements are sometimes
discovered during board bring-up requiring changes. Bodging the board to change the sequence is possible but annoying,
so I'd rather do it in software.

## The plan

After a few iterations I'm starting to converge on using the STM32L431 as a supervisor. It's around $3 in moderate
volume as of this writing depending on package - definitely not the cheapest option but compared to a large Kintex-7 or
UltraScale+ FPGA is an insignificant contribution to system BOM. It's got plenty of horsepower - 80 MHz Cortex-M4, 256
kB of flash, and 64 kB of SRAM - so more than enoguh flash space for storing a bootloader and A/B images for
high-reliability updating. More importantly, it has a wide range of package options (32-QFN, 48-QFN, and 100-BGA)
available. This allows me to trivially port firmware between boards with varying numbers of power domains.

The basic job of the supervisor is to provide soft power on/off capability, sequence power rails up and down during
normal startup and shutdown, and provide monitoring capabilities for system health checks (using its internal ADC and
temperature sensor, as well as connecting to the IBC over I2C to query input power state).

Additionally, the supervisor needs to handle fault cases such as a regulator's PGOOD pin going low (likely indicating a
regulator fault or short), an ADC reading out of the acceptable tolerance range, overheating, or a loss of input power.
Power supply faults result in an immediate hard shutdown of the entire system to avoid  further damage; if power is
lost a warning signal is asserted for a short time so that the system can save any volatile state, followed by a clean
sequenced shutdown of all power rails.

## Current state

The supervisor is functional in various forms and in use across a bunch of prototypes now but the code needs some
cleanup and refactoring to be a reusable module that I can easily throw down in a new project and just plug in a list
of which GPIOs control which power domains, required sequencing, etc. There will be a dedicated post about this
eventually.

At some point I'd like the supervisor to also expand to other system health functions that should be independent of
primary application firmware, like controlling fans.
