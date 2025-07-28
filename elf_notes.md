# Notes about ELF Files

This document mainly serves as an explanation of the stuff being written, to aid in writing it, and to explain why each byte is where it is.
Both as an attempt at documentation, and an attempt to prove that I did actually write this by hand, since there's naturally no source codes.

## ELF Files Overview

The Executable and Linkable Format (ELF) is a common UNIX/Linux executable file type, originally developed for UNIX, now mainly used in Linux.
It can be used for executable binaries, shared objects (libraries) and other binary files.

Being an open specification, there's a lot of technical information accessible online on the specifics of the format.

### Primary information sources

The primary information sources are as follows:

- https://refspecs.linuxfoundation.org/elf/gabi4+/ch4.eheader.html
- https://gist.github.com/x0nu11byt3/bcb35c3de461e5fb66173071a2379779

### ELF Internal File Structure

Each file consists of:

- **ELF Header**: Containing general information about the ELF File
- **Program Header Table**: Table of program headers containing information the OS needs to load the file
  - **Program Header**: Entry describing a segment of the program and how to work with it
    - Mainly LOAD a segment into memory with which permissions at which virtual address
    - Can be a different segment for RW and different for RE permissions for data/instructions
- **Program**: Actual bytecode of the program
  - Consists of bytes, including program instructions and program memory
  - Split into sections, most commonly includes `.text`, `.data`, `.bss` and symbol and string tables
- **Section Header Table**: Table of section headers describing each section of the program
  - **Section Header**: Entry describing a section of the program, where it's to be loaded and what permissions it has

To analyse ELF files in more detail, tools such as `nm` and `readelf` can be used.

### ELF Header

The following is a description of a 64-bit ELF header, including which parts are at what address:

- 0x00-0x03: ELF Magic Number, 0x7F 0x45 0x4C 0x46 (0x7F 'E' 'L' 'F')
- 0x04-0x0F: ELF File Options
  - 0x04: Class, 0x01 for 32-bit and 0x02 for 64-bit
  - 0x05: Data Encoding, 0x01 for LSB, 0x02 for MSB (0x01 default)
  - 0x06: ELF Header Version number, EV_CURRENT is 0x01 (as of 2025)
  - 0x07: OS/ABI-specific extensions, recommended to use 0x00 for none specific
  - 0x08: ABI Version, dependant on 0x07, can be left to 0x00 for unspecified
  - 0x09-0x0F: Padding bytes, set to 0x00
- 0x10-0x11: ELF type: 0x02 for executable file, 0x03 for shared object file
- 0x12-0x13: Machine type: 0xB7 for ARM AARCH64, 0x3E for AMD x86-64 among others
- 0x14-0x17: ELF file version, 0x01
- 0x18-0x1F: Entry point address (64 bits for 64-bit ELF, would be 0x18-0x1B for 32-bit)
- 0x20-0x27: Program Header Table file offset (in bytes, for 64-bit Linux commonly 0x40, if right after header)
- 0x28-0x2F: Section Header Table file offset (bytes)
- 0x30-0x33: Processor-specific flags (can be left 0)
- 0x34-0x35: ELF Header file size (0x40/64 for 64-bit ELF)
- 0x36-0x37: Program Header entry size (all entries same size, for 64-bit Linux ELF it's 0x38/56)
- 0x38-0x39: Number of entries in the Program Header Table
- 0x3A-0x3B: Section Header Table entry size (64-bit Linux ELF is 0x40/64)
- 0x3C-0x3D: Number of entries in the Section Header Table
- 0x3E-0x3F: Section Header Table entry index containing the offset to the String Table

### Program Header

