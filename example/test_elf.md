# Analysis of test.elf

To experiment hands-on and see how ELF files actually work, I wrote the `test.elf` program in a hex editor, to serve as a minimal hello world example.

This accompanying document contains an analysis of all bytes in the file, to convey understanding of the actual binary.

## Obtaining bytes

The command `hexdump -X test.elf` can be used to obtain the bytes of the binary in text format.

The following output can be obtained:

```
0000000  7f  45  4c  46  02  01  01  00  00  00  00  00  00  00  00  00
0000010  02  00  b7  00  01  00  00  00  78  00  40  00  00  00  00  00
0000020  40  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00
0000030  00  00  00  00  40  00  38  00  01  00  40  00  00  00  00  00
0000040  01  00  00  00  07  00  00  00  00  00  00  00  00  00  00  00
0000050  00  00  40  00  00  00  00  00  00  00  40  00  00  00  00  00
0000060  af  00  00  00  00  00  00  00  af  00  00  00  00  00  00  00
0000070  00  00  10  00  00  00  00  00  08  08  80  d2  01  00  80  d2
0000080  c1  00  00  58  62  01  80  d2  01  00  00  d4  a8  0b  80  d2
0000090  00  00  80  d2  01  00  00  d4  a0  00  40  00  00  00  00  00
00000a0  48  69  69  69  21  21  20  3a  33  0a  00  00  00  00  00  00
00000b0
```

Alternatively, opening the file in a hex editor (such as `hexedit`) is helpful as well, since it additionally shows the address of the currently selected byte.


## Byte analysis

To understand what's actually happening here, we can look at analysing the bytes.

### LSB encoding

This file, and most ELF binaries, are encoded using LSB, meaning Least Significant Bit first.

In practice, this means that when reading a number spanning multiple bytes, they are in reverse order to what one would naturally expect.
The characters of the bytes themselves are in the usual order, which is important to keep in mind.

For example, the 4 bytes `0x08 0x08 0x80 0xd2` together create a 4-byte hexadecimal number usually denoted `0xD2800808`.

### ELF header

The first 64 bytes (0-63 or 0x0-0x3F) of the file are the ELF header.

It starts with the magic number, `0x7f 0x45 ('E') 0x4c ('L') 0x4 ('F')`.

Next is `0x02`, the class field, indicating this to be a 64-bit executable.
This is followed by a `0x01` signifying LSB encoding, and another `0x01` being the ELF header version number.
After this comes 9 empty bytes, meaning unset OS-specific extensions, unspecified ABI version and a few padding bytes.

The next byte is a `0x02 0x00`, at address `0x10` and starting the second line of the hexdump output.
This byte means that the file is an executable file, rather than a shared object file (`0x03`).

After this comes `0xb7 0x00` which means the ELF file contains machine code for the Arm64 architecture.
Each architecture has its specific code, for example `0x3E 0x00` for amd64.

The next 4 bytes (`0x01 0x00 0x00 0x00`) are the ELF file version.

The 8 bytes following the version contain the entry point address of the program (`0x78 0x00 0x40 0x00 0x00 0x00 0x00 0x00`).
When contents of this file are loaded into RAM, to a virtual address specified by the program header table, 
this number denotes the virtual address in the loaded program's address space to jump to to begin execution.

The next 8 bytes are the program header table offset in the actual file, usually `0x40 (64)` since that's directly after the size of the ELF header.
The 8 bytes after that are the section header table offset in the actual file.
Since this file contains no section headers, this is 0.

This is followed by 4 bytes of processor-specific flags, left to none in this file.

After follows a field with 2 bytes containing the ELF header size, here `0x40 0x00` for 64.
Directly after are 2 bytes with a program header entry size, on regular Linux this is `0x38 0x00`.
This is followed by the number of entries in the program header table, here `0x01 0x00` since there's a single entry.

After, there are 2 bytes containing the section header entry size, here `0x40 0x00` as is usual for a 64-bit Linux ELF.
Next are 2 bytes containing the number of entries in the section header table, here `0x00 0x00` since there's no section headers.
The last 2 bytes contian the index of the String Table in the section header table, if present (here `0x00 0x00` since there's none).

That concludes the ELF header.

### Program Header Table

The program header table follows the ELF header, and contains information needed for the OS to create a process out of the binary.
In practice, mostly this just means which part of the ELF file should be loaded to where in memory, so that the right parts of the file can run.

The first 4 bytes (`0x01 0x00 0x00 0x00`) signify that this entry of the program header table contains a segment to be loaded (LOAD).

The next 4 bytes (`0x07 0x00 0x00 0x00`) contain permission specifiers, `0x7` being RWX.

The next 8 bytes contain the offset of the loadable segment.
If only a specific part of the file was being loaded, this would be the location in the file where the segment starts.
For simplicity, this ELF just loads the entire file, so the offset is left to 0 (beginning of the binary file).

The next 8 bytes and the 8 bytes following that contain the virtual and physical memory addresses to load the segment to.
For Linux, the addresses are usually the same, though the physical address field is unused and can also be 0.
In this file, the address is set to `0x00 0x00 0x40 0x00 0x00 0x00 0x00 0x00`, or `0x400000` in a more human-readable format.

The address can't be 0 since that would violate null pointer associated constraints the Linux OS imposes.
Choosing `0x400000` as an address places the code to 4 MiB in the program's virtual address space, 
which allows it to avoid potential kernel-reserved parts of the memory, while also allowing proper page alignment with offset 0.

The next 8 bytes are the file size of the current segment, meaning how many bytes from the offset should be loaded to memory.
Here, this is set to `0xAF` which is the size of the ELF file (meaning load the entire file from offset `0x0`).

The 8 following bytes are the memory size of the current segment, meaning into how many bytes in memory the loaded data should be placed.
If the memory size is the same as the file size, this loads the entire file size as expected.
If the memory size is greater, it populates the rest of the memory area with 0s.
If the memory size is smaller, chances are the ELF won't execute properly.

The last 8 bytes of the program header entry are the alignment value.
Due to how the OS works, loadable segments must have congruent address and offset values.
The alignment field is defined to be a power of 2 such that `Offset == Virtual Address mod Alignment`.
Here, this applies, since `0x400000 % 0x100000 == 0x0`.

This concludes the program header entry, and with it the program header table.

### AArch64 Machine Code

What follows are the actual instructions for the very basic hello world program present in the file.

The first instruction, `0x08 0x08 0x80 0xd2`, consists of the bits `11010010100000000000100000001000`.
Upon further analysis, this can be deciphered to be the AArch64 MOV instruction (`11010010100 imm16 (16 bits) Rd (5 bits)`).
The 16 bits of the `imm16` value represent the literal to be moved, and the 5 bits of `Rd` represent which register to move them to.

The first instruction, therefore, moves `0b0000000001000000 (64)` to register `8`.
The next moves the number `1` into register `0`.

This is in preparation of the Write system call.
System calls in Linux are invoked using the `SVC 0` instruction, and take the syscall number from register `8`, and arguments from registers `0` up to `7`.

The Write system call has the ID `64`, and takes a file descriptor in register `0`, a buffer address in register `1` and the buffer length in register `2`.

The first 2 instructions set up registers `8` and `0`, while register `1` is set using the next instruction, LDR (literal).
The instruction `0xc1 0x00 0x00 0x58` consists of the bits `01011000000000000000000011000001`.
This is the AArch64 LDR instruction (`01011000 imm19 (19 bits) Rt (5 bits)`), which loads a word into register `Rt` from the program counter and a literal offset.
The 19 bits of the `imm19` value represent a signed offset, which is multiplied by 4 (bitshift by 2 to the left) and added to the program counter.

Here, the `imm19` value is `0000000000000000110 (0x6)`, which is `0x18` when multiplied by 4.
This means that the value to be loaded is 24 bytes to the right of the current instruction.
Looking at the bytes, we can find the address `0xa0 0x00 0x40 0x00 0x00 0x00 0x00 0x00`, which points to the location after the instructions where our text is.

This is followed by another MOV instruction, which moves the value `11` into register `2`, meaning the length of the text to print.

The next bytes are `0x01 0x00 0x00 0xd4` which are the `SVC 0` instruction, interpreted by Linux to be a system call.

This is followed by instructions setting up and performing the exit system call (ID `93`) the same way as the Write was performed previously.

The instructions themselves are directly followed by a Literal Pool, which is a space containing literal values accessed through LDR with offset.
This was shown previously when loading the address to the hello world text into register `1` for the Write syscall.

The Literal Pool is followed by a component where the actual text is stored.
This would typically be a section known as `.data`, however, since there's no section header table, it's not entirely counted as such.

This concludes the machine code and the entire ELF example file.

