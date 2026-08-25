+++
title= "Memory Management Part 2"
date= "2026-08-24"
+++

Now that we have virtual memory in place, and a way to hand out allocate whole pages, let's see how to allocate memory more granualarly, namely some kind of `malloc`.

## malloc 

`malloc` is a function in the C standard library that allows you to request some bytes of memory. You pass into it an integer denoting how many bytes of memory you
want and it returns a void pointer to the beginning of the block of memory. This memory is allocated on what's called the heap, as opposed to the stack which is 
handled automatically by your compiler, using `push` and `pop` operations. So when you do something like:

```c
int main() {
    void* ptr = malloc(10);
}
```

The variable `ptr` is allocated on `main`'s stack frame, but when we read from that pointer, we'll be looking at heap memory.

## free

`free` is a function that is paired with `malloc`. As the name implies, it deallocates heap memory. You pass into it the pointer that `malloc` returns and the operating system
will make that memory free to use again. Failing to deallocate heap memory may result in [memory leaks](https://en.wikipedia.org/wiki/Memory_leak).

## Implementation

So now we have a rough idea of what `malloc` and `free` do, let's think about how we can achieve something similar in this operating system.

The main data structure that the OS uses to manage heap memory is a free list. This is a type of linked list that we build into the block of heap memory.
It consists of "headers" that contain information about the block of memory.

```c
struct memory_header {
    size_t size;
    struct memory_header *next;
    uint32_t magic;
    bool is_free;
}
```

The size field is how many bytes the following memory block is, the next field points to the next memory block in the free list, the magic number is used as a 
sanity check when freeing memory and is_free denotes the status of the memory block.

Let's say we use a 4096 byte page to use for our heap. We would initialise this page by writing this `memory_header` to the start of it. Because the header
itself uses up some memory, we subtract `sizeof(memory_header)` from 4096 to determine the size field. The next field is NULL, the magic number can be anything
(in my case its 0xDEADBEEF) and the is_free bool is set to true.

Now when we call malloc, we should go to the start of that page and read the header. If `size` is less than the number of requested bytes, n, follow the next pointer.
If the block's size field is exactly equal to n, return a pointer to the start of that block's memory.

Now, if we have a block which is much larger than n, it would be inefficient to give the user the whole block, so we split the block instead. Initialise another 
memory header inside the existing block and update the respective fields, for example:

```c
if (next->size >= (requested_size + sizeof(memory_header))) {
    memory_header *new =
        (memory_header *)((char *)next + sizeof(memory_header) + requested_size);
    new->size = next->size - requested_size - sizeof(memory_header);
    new->is_free = true;
    new->magic = 0xDEADBEEF;
    new->next = next->next;

    next->size = requested_size;
    next->next = new;
    next->is_free = false;
    return (void *)((char *)next + sizeof(memory_header));
}
```

Now imagine if we make lots of small malloc requests. First of all this is inefficient because each time a split is done, `sizeof(memory_header)` bytes are used as overhead.
Second, after we free these blocks, we now have a bunch of small blocks of memory. If we try to malloc a larger number of bytes, we can't use these blocks since
they are too small. The following function walks the free list and combines adjacent free blocks. 

```c
void coalesce() {
    memory_header *curr = (memory_header *)START_ADDR;
    while (curr != NULL) {
        while (curr->is_free && curr->next != NULL && curr->next->is_free) {
            curr->size += curr->next->size + sizeof(memory_header);
            curr->next = curr->next->next;
        }
        curr = curr->next;
    }
}
```

Now if we have the case where there are 4 adjacent, free blocks of 10 bytes, and we need 40 bytes, this function allows us to combine them. 

Finally, an implementation of free:

```c
void kfree(void *src) {
    memory_header *header =
        (memory_header *)((char *)src - sizeof(memory_header));
    if ((uint32_t)header < heap_boundary && (uint32_t)header >= START_ADDR &&
        header->magic == 0xDEADBEEF) {
        header->is_free = true;
        coalesce();
    } else {
        printf("invalid ptr: 0x%p\n", src);
        __asm__ volatile("cli; hlt");
    }
}
```

Here we use the magic number to verify that the pointer that was passed in to free was indeed a pointer from malloc. The variable `heap_boundary` tells us
where the heap ends and gets updated whenever we grow the heap. If the given pointer is invalid, just halt.

## Sources
Operating Systems Three Easy Pieces - [link](https://ostep.org/)
