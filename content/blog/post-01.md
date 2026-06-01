+++
title= "Part 1 - Setup and Serial Output"
date= "2026-06-01"
+++

To start off this OS project, I followed some tutorials on the OSDev wiki. This got me:

- A [cross-compiler](http://wiki.osdev.org/GCC_Cross-Compiler) that targeted i386 (a 32-bit x86 architecture)
- A [template](https://wiki.osdev.org/Meaty_Skeleton) which organised the files of the project (including Makefiles and convenience shell scripts)
- A black screen that prints "Hello, kernel World!"

The template gave me a kernel that boots via GRUB using the Multiboot standard, a small libc (compiled as `libk` for kernel use), and a VGA text-mode terminal driver. The build system uses a cross-compiler (`i686-elf-gcc`) with a sysroot, so the kernel and libc are built in a freestanding environment -- no host OS headers or libraries.

## Serial Output

The first thing I wanted was a way to get debug output on my host terminal, not just the screen inside the emulator. QEMU takes a `-serial stdio` flag that 
maps the emulated serial port to the host's stdin/stdout.

Before we can use the serial port, we need port I/O. On x86, hardware is accessed through I/O ports using the `in` and `out` instructions. I used some wrapper 
functions around inline assembly for `inb` (read a byte from a port) and `outb` (write a byte to a port).
(see [this](https://wiki.osdev.org/Inline_Assembly/Examples#I/O_access))

Initialising the serial port involves writing a specific sequence of bytes to configure the serial communication.
With the base address of COM1 `0x3F8`, we can configure several settings by adding an offset to it in order to access the registers of the UART chip.

After initialisation, sending a character is just a matter of waiting for the transmit buffer to be empty (polling bit 5 of the Line Status Register at port 
`0x3FD`), then writing the byte to the data port. I wrapped this up with a `write_serial_string` function to send whole strings.

Now I could call `write_serial_string("Hello, host!\n")` from my kernel and see it appear in my terminal.

## Sources
[Serial Ports](https://wiki.osdev.org/Serial_Ports)
