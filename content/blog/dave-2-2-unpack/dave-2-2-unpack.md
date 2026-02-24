---
title: Unpacking and Disassembling Dave 2
description: Taking our core executable and starting the disassembly.
date: 2026-02-24
tags: engineering,reverse-engineering,ms-dos,dave-2,gaming,ghidra
---

So <a href="../dave-2-port/dave-2-port.md">in the last post</a> we got stuck
looking at an executable full of garbled strings.  In this post I’ll walk
through how I got past that hurdle and began disassembling the executable. ##

## Our first Rosetta Stone

> It would not be our last.

So while searching through these strings, one string in particular stood out:

```text
44 61 6e 67 2a 6f 75 73
D  a  n  g  *  o  u  s
```

Given the game title, this looked like it should encode the string "Dangerous",
but how could a single byte represent the string "er" when all other surrounding
bytes are regular ASCII?

This was a strong clue that some sort of compression was involved.

### Self extracting executables

A bit more research revealed that it was common in this era to ship
self-extracting executables, presumably because shipping on physical media was
so constrained.

Taking another look in the hex editor, I spotted the string `LZ91` at offset
0x1c within the MZ header. This gave me a lead that quickly led to
documentation on the [`LZEXE`](https://moddingwiki.shikadi.net/wiki/LZEXE)
utility, and its inverse,
[`UNLZEXE`](https://keenwiki.shikadi.net/wiki/UNLZEXE).

So armed with a 16-bit version of `UNLZEXE` I fired up DOSBox and ran it over
`DAVE.EXE`.  Lo and behold, our executable now has strings:

<img src="./strings.png" alt="Glorious strings" />

This was my first real win on this project, and it no longer seemed like purely
an academic exercise.  I avoided getting side-tracked here into the details of
specifically how this extraction worked in practice, and decided to proceed
under the assumption that we could get to work reverse engineering the output
executable.

## Decompilation

So strings are one thing, but it was time to bring some new tooling to the table
and step into the world of decompilation.

I selected [Ghidra](https://ghidra-sre.org/) first, mostly because it seemed
fully featured, well supported and able to decompile 16-bit MS-DOS executables.

I wasn't sure what to expect. There would be no symbols and very little
structure. I’d [dabbled enough with
emulation](https://github.com/mathewbyrne/chip-8) and remembered enough
computing theory to follow assembly, but I didn’t yet have a clear plan. At this
stage I was still stumbling around, unsure what was realistically achievable.

<img src="./ghidra-import.png" alt="Ghidra detects executable type" />

Importing the unpacked executable is daunting, but gave me a lot more confidence
that I was on the right track.  Immediately I could see several hundred
functions, each named purely by its address in memory.

For a DOS game written by a small team in a couple of months, a few hundred
functions seemed plausible.  Even then, scanning the disassembly suggested there
were large regions that might yet contain additional functions.

<div class="callout">
	<div class="callout__title">Note</div>
	<div class="callout__body">
		<p>I am definitely no expert in either MS-DOS programming nor Ghidra,
		so take any content from here on forward as an exploration of these
			topics.  I'll do my best to link to better sources where possible.</p>
		<p>If you have any corrections or additional detail, I'd love to hear from
			you!</p>
		</div>
</div>

Ghidra immediately detected a 16-bit MS-DOS MZ executable[^1].  It also marked
something called [_Real Mode_](https://en.wikipedia.org/wiki/Real_mode), which
relates to the 16-bit segmented addressing model used by MS-DOS.

And once it's done an import, and an initial analysis with default settings, we
get left with a screen full of assembly, some very obfuscated looking C
decompilation window, and 373 functions.

Above the first instruction, Ghidra is indicating some assumptions around what
appear to be registers.

```asm
//
// CODE_0
// ram:1000:0000-ram:1000:d98f
//
**************************************************************
*                          FUNCTION                          *
**************************************************************
undefined __stdcall16near entry(void)
  assume CS = 0x1000
  assume DS = 0x3713
  assume SP = 0x80
  assume SS = 0x448c
  <UNASSIGNED>   <RETURN>
entry                      XREF[1]:     Entry Point(*)
    MOV        DX,0x3713
```

Immediately I notice the memory addresses, `1000:0000` is the location of the
first instruction and a bit of research shows that this is a [segmented memory
address](https://en.wikipedia.org/wiki/X86_memory_segmentation), with the
segment `0x1000` and an offset of `0x0000`.

My first instinct here is to try and understand a little bit about how MS-DOS
would execute this file.  I guess that Ghidra was mapping the executable into a
synthetic memory layout for analysis and appeared to choose `0x1000` as an
arbitrary base for mapping the load image into memory. This isn’t a DOS
requirement — it’s simply a convenient paragraph-aligned segment used for
analysis.

My next instinct was to understand a little more about Ghidra.  Fortunately they
provide some pretty nice educational content in their repository.  Their
[Introduction to
Ghidra](https://html-preview.github.io/?url=https://github.com/NationalSecurityAgency/ghidra/blob/master/GhidraDocs/GhidraClass/Beginner/Introduction_to_Ghidra_Student_Guide.html)
class was particularly useful for getting started understanding the suite of
features on offer.

## Labels and Types

Being able to assign labels and types immediately stood out as features I could
begin using to get a foothold in the codebase.

I quickly discovered that the decompiled C is not the source of truth — it’s
Ghidra’s best attempt to express the underlying assembly in a higher-level form.
I had assumed that generally there would be more lines of assembly than C, and
while this is usually true, it's also true that seemingly simple sets of
instructions can map to multiple lines of C code.

Ghidra provides a lot of tooling for labelling and annotating different parts of
the application.  It also supports being able to type memory blocks with both C
primitives, but also defined structs.  Ghidra does surprisingly well at
inferring the size and structure of memory regions in use, and once you can
correctly assign types to memory and registers, the decompilation can get quite
close to something you could compile.

## One Battle After Another

With Ghidra now up and running, and the basics of the codebase seemingly mapped
out it was time to start forming a game plan, and it probably looked something
like this:

1. Explore the executable, labelling and typing anything we can so that we can
   start to understand the program logic and flow.
2. As we begin understanding different subsystems, we start building modern
	 reimplementations — probably assets first, then graphics, sound, input, etc.
3. Locate the main game loops and start structuring our application to
	 replicate.
4. Understand gameplay systems and start reimplementing those — player movement,
	 entity management, collision.
5. Continue repeating these steps until we have a working reimplementation and
	 have exhausted or documented all the code paths.

At this point I still don’t have a good understanding of the scope of this work,
or how long it will take. But each session I spend tracing through the assembly
and understanding what I'm looking at, I make a little more progress.

I've started a [codebase here](https://github.com/mathewbyrne/dave-2-port) where
I'm beginning to implement the above.  From here on out the articles will
probably be focussed on specific subsystems that I discover, how I traced them,
and how I plan to implement them.

Next up, I’ll start tracing the game’s asset loading routines and begin writing
our own loader.

[^1]: https://wiki.osdev.org/MZ
