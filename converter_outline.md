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
With alignment to `0x1000` we can place the segment to the VRAM address `0x4000B0`, to clear enough space beforehand for OS/kernel things.
A small alignment value is chosen to ensure the executable uses as little RAM as possible.

If we pad the first segment to be 512 bytes in the executable itself as well, it can be at offset `0x2B0`.
This allows us to place it into VRAM directly after the first, at address `0x4012B0` aligned to `0x1000`.
The address is `0x4012B0` rather than `0x4002B0` as per the alignment, to ensure the segments aren't mapped to the same virtual memory.
The content itself of the second segment is (based on the known program strings) 141 bytes, with an additional single zeroed bss byte at the end.

This leads to the segments being the following:

- **Segment 1:**
  - PT_LOAD (`0x1`), R-E (`0x5`)
  - Offset: `0xB0`, Virtual/Physical Address: `0x4000B0`
  - File size: `0x200` (512), Memory size: `0x200` (512), Alignment: `0x1000`
- **Segment 2:**
  - PT_LOAD (`0x1`), RW- (`0x6`)
  - Offset: `0x2B0`, Virtual/Physical Address: `0x4012B0`
  - File size: `0x8D` (141), Memory size: `0x8E` (142), Alignment: `0x1000`

The entry point to the program would thus be the address `0x4000B0`.

## Notes on some program instructions

**Stack Usage:**

To use the stack, we can use the usual `add`/`sub` instructions.
The stack pointer (sp) in AArch64 is, in instructions that support the stack, the 32nd register, `0b11111`.

**Mov instruction for system calls:**

Syscall setup requires a few `mov` instructions, to registers `x8` for the system call ID and `x1, x2...` for the system call arguments.
The IDs used here will be `64` (write), `63` (read) and `93` (exit).
For the `mov` instruction itself, as per the 
[Arm developer reference](https://developer.arm.com/documentation/ddi0596/2020-12/Base-Instructions/MOV--wide-immediate---Move--wide-immediate---an-alias-of-MOVZ-), 
the `sf` flag is to be set to 1 for 64-bit, and the `hw` flag is to be set to `00` (no shitf).

**Address Pool:**

To save a 64-bit memory address into a 64-bit instruction, an address pool must be used.
This is because the instruction width is 4 bytes, which is shorter than the 8 bytes necessary for an address,
and being a RISC architecture, AArch64 has a consistent instruction width.

The address pool can be accessed using the LDR (literal) instruction, available here in the
[Arm developer reference](https://developer.arm.com/documentation/ddi0596/2020-12/Base-Instructions/LDR--literal---Load-Register--literal--).
Here, x is set to 1 to signify 64 bits, making the instruction `01011000 imm19 Rt`.
The `imm19` value here contains a signed offset from the PC at which the word to load is found.
The offset value should be the number of instructions rather than the number of bytes, as the CPU itself multiplies it by 4.

**Stack Pointer Mov:**

Moving the stack pointer address into a regular register is done using a different instruction than a MOV (register).
The specific instruction MOV (to/from SP) has documentation which can be found at the
[Arm developer reference](https://developer.arm.com/documentation/ddi0602/2020-12/Base-Instructions/MOV--to-from-SP---Move-between-register-and-stack-pointer--an-alias-of-ADD--immediate--).
As mentioned before, the stack can be represented as `0b11111` here.

**Branch Instructions:**

The standard branch instruction jumps to a PC-relative signed offset, with the format `000101 imm26`.

Conditional branch instructions first require a CMP instruction (which is fairly straightforward).
The B.cond instruction itself contains a field called `cond` which encodes the condition, as per the
[Arm developer reference](https://developer.arm.com/documentation/ddi0596/2020-12/Base-Instructions/B-cond--Branch-conditionally-?lang=en).
This is one of the standard conditions, encoded in the standard way.
The standard encodings can be found online, for example [here](https://finkmartin.com/aarch64/b_cond.html).

**Load Byte:**

To load a byte from a memory address with a register-specified offset, LDRB (register) can be used, as per the
[Arm developer reference](https://developer.arm.com/documentation/ddi0596/2020-12/Base-Instructions/LDRB--register---Load-Register-Byte--register--).
The option flag is set to 011 for shifted rather than extended register.
The `S` option is set to 0 for no shift amount.

**Store Byte:**

To store a byte to a memory address with a register-specified offset, STRB (register) can be used, as per the
[Arm developer reference](https://developer.arm.com/documentation/ddi0596/2020-12/Base-Instructions/STRB--register---Store-Register-Byte--register--).
The option flag is set to 011 for shifted rather than extended register.
The `S` option is set to 0 for no shift amount.
This instruction works basically the same way as the aforementioned LDRB (register).

**Logical Shift Left:**

The LSL (immediate) instruction is an alias for UBFM, the unsigned bitfield move, as per the
[Arm developer reference](https://developer.arm.com/documentation/ddi0596/2020-12/Base-Instructions/LSL--immediate---Logical-Shift-Left--immediate---an-alias-of-UBFM-)
In binary, this is represented as an UBFM instruction with a few specific settings.
The `sf` flag and the `N` flag are set to 1 to signify 64 bits.
The `immr` field is set to `-shift MOD 32` and the `imms` field is set to `63 - shift`, given a value `shift` to shift by.

## Notes on bytecode

The flowchart displayed above outlines how the binary is split up into sections.
The following is a brief description of which parts are where in the byte code itself.
The addresses written here are the offsets of the first of the 4 bytes of each instruction within the `.elf` file.

- `0xB0` ............ Allocating 64 bits on stack
- `0xB4`-`0xC4` ..... Printing introduction text
- `0xC8`-`0xD4` ..... Printing text asking for binary number
- `0xD8`-`0xE8` ..... Reading user input containing binary number
- `0xEC`-`0x120` .... Convert read bytes into register
- `0x124`-`0x150` ... Print register to decimal bytes
- `0x154`-`0x170` ... Store printed register and print 'the number is' text
- `0x174`-`0x180` ... Get printed number register and print to stdout
- `0x184`-`0x190` ... Print prompt asking user to convert again or not
- `0x194`-`0x1A4` ... Read user input containing y/n answer
- `0x1A8`-`0x1AC` ... Load read user input byte and set up for next write
- `0x1B0`-`0x1B4` ... Check if user input is 'Y' and branch to `0xC8` if so
- `0x1B8`-`0x1BC` ... Check if user input is 'y' and branch to `0xC8` if so
- `0x1C0`-`0x1C8` ... Exit with code 0 if this part reached

The remaining parts are as follows:

- `0x1CC`-`0x27F` ... Padding until next part, null bytes
- `0x280`-`0x2AF` ... Local literal/address pool for LDR (literal) instructions
- `0x2B0`-`EOF` ..... Data segment containing strings

