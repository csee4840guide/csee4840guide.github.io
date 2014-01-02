---
layout: default
title: Creating Megafunctions using MegaWizard
---

Altera megafunctions are predefined hardware IP modules which you can use in
your design. While you could write most of these megafunctions yourself in VHDL,
using megafunctions can save you a lot of time. Also, megafunctions offer the
most straightforward way to use the dedicated FPGA hardware, such as PLLs,
M4K block RAM, and hard multipliers. The guide will focus on these three
functions.

You can add megafunctions to your design using the MegaWizard plug-in manager,
which you can find in the "Tools" menu.

## PLLs

A PLL is a phase-locked loop. It is a method of generating a clock signal with
very low jitter (variations in frequency). You will need to use PLLs if you
want to interface with hardware peripherals such as the audio codec and VGA,
which operate on clocks slower than the native 50 MHz clock. You can find
the PLL megafunction in MegaWizard under "PLL" -> "Altera PLL". The options
for this unit are relatively intuitive. It takes a single input clock and can
generate three output clocks of different phases and frequencies.

## RAMs and ROMs

The M4K block RAM on the Cyclone II can be used to create RAMs and ROMs.
This is much more efficient than trying to create equivalent units in VHDL,
as that will cause a lot of logic elements to be used. The Cyclone II has
a decent amount of block RAM, so try to use that instead of SRAM for your
hardware's memory needs, freeing the SRAM to be used by the NIOS II processor.

You can find the RAM and ROM megafunctions in MegaWizard in the "Memory Compiler"
folder. The synchronous RAMs and ROMs will add a flip-flop to all of the inputs
and optionally to the outputs. With flip-flops on the input only, there will be
a single-cycle delay from the time the read address is changed to when the
read data appears on the output. There will be a two-cycle delay if flip-flops
are placed on the output as well. Your design will have to account for these
delays.

When creating a ROM, you will need to specify a Memory-Initialization File
(.mif) or Intel Hex File (.hex), which holds the data for the ROM.
MIF files are more human readable. You can find the format specification in
the [Quartus Help](http://quartushelp.altera.com/13.1/mergedProjects/reference/glossary/def_mif.htm).
Intel Hex files are easier to generate programatically. In fact, you can
generate them from flat binary files or ELF executables using the `objcopy`
program provided by GNU binutils.
You can find the specification for Hex files on [Wikipedia](https://en.wikipedia.org/wiki/Intel_hex).

## Hardware Multipliers

The Cyclone II has a handful of 9-bit by 9-bit multipliers. Two of these hard
multipliers can also be chained to form an 18x18 bit multipler. You can find
the multiplier megafunction under "Arithmetic" -> "LPM\_MULT". You can specify
any input or output width, but this guide advises against choosing any input
width larger than 18 bits.
