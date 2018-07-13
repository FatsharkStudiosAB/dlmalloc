# Description
A modified version of dlmalloc with support for multiple contiguous heaps withing single proccess.
We use dlmalloc's mspaces as isolated heaps for different subsystems in Stingray game engine. The biggest problem with this approach is that heaps are non-contiguous and as a result of this - a lot of memory is wasted.
We can take advantage of virtual address space to fix this, especialy if you are running x64 proccess (and you should, it's 2018 at the time of writing, and no one should write 32 bit mainstream software any more!). Reserve a lot of memory for your heap and commit and de-commit it on demand from dlmalloc, as simple as that.

# How to use it
Use it as you'd use dlmalloc normally, but add define `MSPACES_CONTIGUOUS` to 1 and define `MORECORE` to your routine that will extend your heap i.e. commit reserved pages for it.
`create_mspace` and `create_mspace_with_base` now accept a pointer to user data that will help you to identify your name space in morecode\mmap\munmap callbacks.

# Example
```C
#define MSPACES                 1
#define MORECORE_CONTIGUOUS     1
#define HAVE_MORECORE           1
#define MSPACES_CONTIGUOUS      1

void* dlmorecore(intptr_t size, void* heap);
#define MORECORE                dlmorecore

#include "dlmalloc.c"

void* dlmorecore(intptr_t size, void* user_data) {
  Heap* heap = (Heap*)user_data;
  void* old_sbrk = heap->_sbrk;

  if (heap->_commited + size > heap->_reserved) {
    return (void*)MAX_SIZE_T;
  }

  if (size > 0) {
    commit_vmemory(heap->_sbrk, size);
    heap->_sbrk = (char*)heap->_sbrk + size;
  } else if (size < 0) {
    heap->_sbrk = (char*)heap->_sbrk + size;
    decommit_vmemory(heap->_sbrk, -size);
  }

  heap->_commited += size;
  return old_sbrk;
}
```
That's it!
It decreased memory consumption in Vermintide 2 by up to 13% or up to 300MB and that is very helpful for systems with memory shortage.
