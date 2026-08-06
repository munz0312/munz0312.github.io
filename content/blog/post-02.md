+++
title= "Memory Management Part 1"
date= "2026-08-06"
+++

It's been a while since my last post, but there has been decent progress since. In this post, I will be talking about how my operating system does memory management.


## Detecting Memory 

The bootloader I used, GRUB, supports the Multiboot standard which includes a way of detecting how much RAM the machine (or, in this case, emulator) has. It provides a memory map which can be viewed in the GRUB menu's command line via the `lsmmap` command.

![Output of the lsmmap command in the GRUB command line, listing base addresses, lengths and whether each region is available or reserved RAM](/blog/lsmmap.png)

However, to use this information in our C program, we need to read it from a pointer that the bootloader gives us. The various structs are defined in the `multiboot.h` header file, in particular we will be using `multiboot_info_t` type.

Using some code I found on the OSDev Wiki [link](https://wiki.osdev.org/Detecting_Memory_(x86)#Memory_Map_Via_GRUB):
```
/* Loop through the memory map and display the values */
int i;
for(i = 0; i < mbd->mmap_length; i += sizeof(multiboot_memory_map_t)) 
{
    multiboot_memory_map_t* mmmt = 
        (multiboot_memory_map_t*) (mbd->mmap_addr + i);

    printf("Start Addr: %x | Length: %x | Size: %x | Type: %d\n",
        mmmt->addr, mmmt->len, mmmt->size, mmmt->type);

    if(mmmt->type == MULTIBOOT_MEMORY_AVAILABLE) {
        /* 
        * Do something with this memory block!
        * BE WARNED that some of memory shown as availiable is actually 
        * actively being used by the kernel! You'll need to take that
        * into account before writing to memory!
        */
    }
}
```

Now I have access to each section of memory, I can divide it up into 4096-byte page frames. To track the status of these page frames, I define a bitmap where a page is used if its corresponding bit is set.
The bitmap is initialised by setting all bits, then whilst iterating over the sections of availiable memory, we can clear the bits to mark it as available.

## Virtual Memory & Paging

Virtual memory is an abstraction over the physically availiable memory. Every process that runs is under the illusion of having a fully available,  addressable memory space. This is known as its virtual address space.
Page tables are data structures that reside in memory which contain virtual to physical memory mappings.

In my operating system, I implement 2 levels of page tables. The top level page table, which I will refer to as the page directory, contains page directory entries which each point to a page table.
The page tables then contain the physical address of the page that corresponds to the virtual address.

When paging is enabled, the 10 MSBs of a given 32-bit virtual address is used to index into the page directory, the next 10 MSBs are used to index into the page table, and the final 12 bits are the offset into the page.

## Recursive Page Table Mapping

This was a concept that I found difficult to understand, but I'll try my best to explain the problem and its solution. As mentioned, after enabling paging, we now use virtual addresses. These virtual addresses are mapped to physical ones through the page directory and page tables.
But what if we want the address of the page directory/page tables - perhaps to extend/edit them? 
We can't use their physical address, because virtual memory is enabled.
And we can't create another page table for the page tables, because the problem still exists - how would we then access those page tables?
The solution is to take one page directory entry (in this case, PDE[1023]) and have it point to the page directory.

So now, if I wanted access to the physical page that contains our page directory, I would use the virtual address `0xFFFFF000`. In binary, the 10 MSBs are `0b1111 1111 11` which is 1023.
So we access page directory entry 1023. This entry points us back to the start of the PD. Then the next 10 MSBs are also `0b1111 1111 11`. This again gets us to PDE[1023], which again points us back to the start of the PD.
The 12 LSBs give us an offset of 0, so we stay at the start of the PD, and the virtual-to-physical address translation is done and we have access to our PD, free to edit it.


## Sources
Stuff about recursive page tables - [link](https://wiki.osdev.org/User:Neon/Recursive_Paging)

