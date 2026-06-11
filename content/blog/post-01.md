+++
title= "Part 1"
date= "2026-06-01"
+++

To start off this OS project, I followed some tutorials on the OSDev wiki. This got me:

- A [cross-compiler](http://wiki.osdev.org/GCC_Cross-Compiler) that targeted i386 (a 32-bit x86 architecture)
- A [template](https://wiki.osdev.org/Meaty_Skeleton) which organised the files of the project (including Makefiles and convenience shell scripts)
- A black screen that prints "Hello, kernel World!"

The template gave me a kernel that boots via GRUB using the Multiboot standard, a small libc (compiled as `libk` for kernel use), and a VGA text-mode terminal driver.

## Serial Output

The first thing I wanted was a way to get debug output on my host terminal, not just the screen inside the emulator. QEMU takes a `-serial stdio` flag that 
maps the emulated serial port to the host's stdin/stdout.

Before we can use the serial port, we need port I/O. On x86, hardware is accessed through I/O ports using the `in` and `out` instructions. I used some wrapper 
functions around inline assembly for `inb` (read a byte from a port) and `outb` (write a byte to a port).
(See [this](https://wiki.osdev.org/Inline_Assembly/Examples#I/O_access))

Initialising the serial port involves writing a specific sequence of bytes to configure the serial communication.
With the base address of COM1 `0x3F8`, we can configure several settings by adding an offset to it in order to access the registers of the UART chip.

After initialisation, sending a character is just a matter of waiting for the transmit buffer to be empty (polling bit 5 of the Line Status Register at port 
`0x3FD`), then writing the byte to the data port. I wrapped this up with a `write_serial_string` function to send whole strings.

Now I could call `write_serial_string("Hello, host!\n")` from my kernel and see it appear in my terminal.

## Global Descriptor Table

The Global Descriptor Table (GDT) is a data structure on x86 that contains information about memory segments - the base and limits of each segment. However, 
I don't plan on using segmentation to separate memory into protected areas, instead I will be using paging.

So, after following this [tutorial](https://wiki.osdev.org/GDT_Tutorial), I had 3 items in the GDT: the null descriptor, kernel mode code segment and kernel
mode data segment. Give that I'm not using segmentation to separate memory, the base and limits for each segment were `0` and `0xFFFFF` respectively. Then, set
the granularity flag so that the limit is in 4KB blocks which is our page size. Therefore we have `0xFFFFF * 4KB = 4GB` addressable space.

I defined the table in C (`gdt.c`) and loaded its address into the GDT register with the `lgdt` assembly instruction.

## Interrupt Descriptor Table

Interrupts, put simply, are a way of alerting the CPU that something needs attention. The IDT tells the CPU of the various routines to be run on the different
kinds of interrupts. Setting it up involves defining a bunch of IDT entries, called gates, then defining what should be done for each interrupt.

In my code, I defined two assembly macros for the two different cases where an interrupt does or doesn't push an error code onto the stack. Then, I push the
interrupt number onto the stack to be used by an external C function `isr_dispatch` defined in (`idt.c`) that does the handling. These macros are then used 
to generate a bunch of the interrupt stubs (`isr_stubs.S`). One of the things I wanted to do with interrupts was keyboard support, but this requires more setup
(Programmable Interrupt Controller).


## Sources
[Serial Ports](https://wiki.osdev.org/Serial_Ports)

[GCC Cross-Compiler](http://wiki.osdev.org/GCC_Cross-Compiler)

[Meaty Skeleton](https://wiki.osdev.org/Meaty_Skeleton)
