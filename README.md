# Binary Converter

This project contains a simple command line tool to convert binary numbers read from the user into decimal strings.
Written entirely, directly in binary.

Created manually, byte by byte, in a hex editor, because I have normal and usual hobbies I promise.

Written for Linux, because I like ELF <3 and for AArch64 because it's simply a better architecture, with no bias whatsoever from my side.

## Repository content

The repo contains the file itself, called `converter.elf`, with an attached commit history.
Sadly, the GitHub UI can't display diffs for this file, however, older versions can be downloaded and compared manually, for example with `hexdump`.

Next to this file is a file called `converter_outline.md`, which contains notes on the converter binary plan, outline of its functionality,
as well as some notes on the implementation, including the specifics of converting some select instruction into bytecode manually.

Along with this, a file called `elf_notes.md` is provided, with an outline of how ELF files in general work, including the ELF header and program headers.

Lastly, the files that were in the repository first are ones present in the `hello_world_example` file.
As a proof of concept, before writing the converter itself, I wrote a simple hello world program, to make sure writing stuff in bytecode is feasible.
This, along with a detailed analysis of each individual byte (in `hello_world_example/hello_elf.md`) can be found in the directory.

