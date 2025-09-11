# Converter Executable Outline

This document mainly contains an outline of the structure for the converter executable, to help with actually writing it.

The Converter executable will read in user input from the command line, more specifically a binary number.
Then, it will parse the bitstring and print it as a decimal number string.
Lastly, it will prompt the user and repeat the process if desired.

The executable will be written in hexadecimal AArch64 machine code in a hexeditor.

## Program Flow Chart

The following is an approximate diagram describing program flow.

```
            /-------\
            | START |
            \-------/
                |
     |---------------------|
     | Allocate 64 B on sp |
     |---------------------|
                |
     |-------------------------|
     | Syscall: Print STRING_1 | <-------- STRING_1 (.data, user welcome text)
     |-------------------------|
                |
     |-------------------------|
 /-->| Syscall: Print STRING_2 | <-------- STRING_2 (.data, user prompt)
 |   |-------------------------|
 |              |
 |   |--------------------------|
 |   | Syscall: Read 64 B to sp | -------> (sp, sp+64)
 |   |--------------------------|
 |              |
 |   |------------------------------|
 |   | Convert bytes at (sp, sp+64) | <--- (sp, sp+64)
 |   | byte-by-byte to a register   |
 |   |------------------------------|
 |              |
 |   |------------------------------|
 |   | Convert number from register | ---> ADDRESS_1 (.data, number string with newline at the end)
 |   | to a decimal string number   |
 |   |------------------------------|
 |              |
 |   |-------------------------|
 |   | Syscall: Print STRING_3 | <-------- STRING_3 (.data, result prompt)
 |   |-------------------------|
 |              |
 |   |--------------------------|
 |   | Syscall: Print ADDRESS_1 | <------- ADDRESS_1 (.data, number string with newline at the end)
 |   |--------------------------|
 |              |
 |   |-------------------------|
 |   | Syscall: Print STRING_4 | <-------- STRING_4 (.data, user prompt whether to calculate again)
 |   |-------------------------|
 |              |
 |   |-------------------|
 |   | Syscall: Read 1 B | --------------> ADDRESS_2 (.bss, 1 byte to hold y/n user answer)
 |   | to ADDRESS_2      |
 |   |-------------------|
 |              |
 |   |--------------------------|
 |   | Load byte from ADDRESS_2 | <------- ADDRESS_2 (.bss, 1 byte to hold y/n user answer)
 |   |--------------------------|
 |              |
 |   |--------------------------\
 | Y | Branch if loaded byte is |
 \---| 'Y' or 'y'               |
     \--------------------------|
                | N
     |---------------------------|
     | Syscall: Exit with code 0 |
     |---------------------------|
                |
             /-----\
             | END |
             \-----/
```

Further analysing each block in the Flowchart, this comes out to about 90 instructions expected in total.

Along with this, we need to use 6 addresses - `ADDRESS_1`, `ADDRESS_2` and 4 strings.
This will need to be stored in an address pool at the end of the instruction segment, and will take an additional 48 bytes.

Since an instruction has 4 bytes, this will lead to 360 B needed in total for instructions.
If we add 48 bytes, this leaves us with potentially 408 B necessary space.
Before writing the program, we also can't exactly guarantee precisely 90 instructions, so it's good to leave a small margin.

Knowing this, we optimally want to allocate something like 512 bytes for the entirety of the instruction segment.

## Program Strings

The following are the strings printed by the program, as presented in the flowchart:

- `STRING_1` ... "Binary Converter Program v1.0\n"
- `STRING_2` ... "Enter binary number (up to 64 bits): "
- `STRING_3` ... "The number in decimal is: "
- `STRING_4` ... "Convert again? (y/N): "

The data at other mentioned memory addresses is:

- `ADDRESS_1` ... 20 empty spaces, followed by a newline and nullbyte
- `ADDRESS_2` ... single zeroed byte

## Segment and Memory Overview

Before writing the executable, it's good to have an idea of the segments, their sizes, offsets and virtual memory addresses.

This is why, for a project such as this, mapping out the planned program is important, since it allows us to know more about segments before the program is written.

The executable will have 2 segments, one with instructions and one with data.
Adding the size of the ELF header with the size of 2 program headers, this leads to the first segment offset being `0xB0`.
With alignment to `0x100000` we can place the segment to the VRAM address `0x4000B0`, to clear enough space beforehand for OS/kernel things.

If we pad the first segment to be 512 bytes in the executable itself as well, it can be at offset `0x2B0`.
This allows us to place it into VRAM directly after the first, at address `0x4002B0` aligned to `0x100000`.
The content itself of the second segment is (based on the known program strings) 141 bytes, with an additional single zeroed bss byte at the end.

This leads to the segments being the following:

- **Segment 1:**
  - PT_LOAD (`0x1`), R-E (`0x5`)
  - Offset: `0xB0`, Virtual/Physical Address: `0x4000B0`
  - File size: `0x200` (512), Memory size: `0x200` (512), Alignment: `0x100000`
- **Segment 2:**
  - PT_LOAD (`0x1`), RW- (`0x6`)
  - Offset: `0x2B0`, Virtual/Physical Address: `0x4002B0`
  - File size: `0x8D` (141), Memory size: `0x8E` (142), Alignment: `0x100000`

