# Heap-Allocator
# Learning Goals
This assignment gives you a chance to implement a core piece of functionality that you've relied on all quarter - a heap allocator! This assignment will help you:
- appreciate the complexity and tradeoffs in implementing a heap allocator
- further develop your pointer and debugging skills
- bring together all of your CS107 skills and knowledge
- you will implement two different heap allocator designs; an implicit free list allocator and an explicit free list allocator. This assignment leaves room for you to experiment and decide between different possible approaches for implementing these allocators to balance various tradeoffs - beyond the requirements listed in this spec, you are free to design your allocators in the best way you see fit!
# Allocator Scripts
- An allocator script is a file that contains a sequence of requests in a compact text-based format. The three request types are a (allocate) r(reallocate) and f (free). Each request has an id-number that can be referred to in a subsequent realloc or free.
a id-number size
r id-number size
f id-number
- A script file containing:
a 0 24
a 1 100
f 0
r 1 300
f 1
is converted by the test harness into these calls to your allocator:
void *ptr0 = mymalloc(24);
void *ptr1 = mymalloc(100);
myfree(ptr0);
ptr1 = myrealloc(ptr1, 300);
myfree(ptr1);-
#### We provide a variety of different test scripts in the samples folder, named to indicate the "flavor":
- example scripts have workloads targeted to test a particular allocator feature. These scripts are very small (< 10 requests), easily traced, and especially useful for early development and debugging. These are much too small to be useful for performance measurements.
- pattern scripts were mechanically constructed from various pattern templates (e.g. 500 mallocs followed by 500 frees or 500 malloc-free pairs). The scripts make enough requests (about 1,000) that they become useful for measuring performance, albeit on a fairly artificial workload.
- trace scripts contain real-world workloads capturing by tracing executing programs. These scripts are large (> 5,000 requests), exhibit diverse behaviors, and useful for comprehensive correctness testing. They also give broad measurement of performance you might expect to get "out in the wild". If you have major efficiency issues, these scripts may not complete – if you’re noticing these scripts never seem to give you information, try running with the pattern scripts to optimize first!
- Your allocator should have no operational or execution errors on any of the provided sample scripts. You are also highly encouraged to create your own!
#### Implementing Your Own Allocators
- Now it's your turn! Your main task on this assignment is to implement your own implicit and explicit allocators as discussed in class and in the textbook. The B&O textbook chapter 9.9 contains background reading on heap allocators, including sample code. You are allowed to review this code, but note that its structure is somewhat incompatible with our framework. Moreover, its code is not that readable and it uses preprocessor macros (#defines) that behave like functions - which you should not use. You can use this code as a starting point, and then write your own. If you do end up taking some inspiration from this code, you may do so given an appropriate citation.
-As you work, make sure to set your own target milestones/checkpoints and implement it _in small, manageable chunks, testing thoroughly at each step. We recommend that you do the code studies first to understand how the test harness works, then work through implicit free list implementation. Finally, port everything over to your high-throughput, high-utilization explicit free list implementation - you should start implementing your explicit allocator with your implicit code, which should automatically pass all tests! Then you can focus on adding the explicit allocator features on top of this.
#### General Requirements
The following requirements apply to both allocators:
- The interface should match the standard libc allocator. Carefully read the malloc man page for what constitutes a well-formed request and the required handling to which your allocator must conform. Note there are a few oddballs (malloc(0) and the like) to take into account. Ignore esoteric details from the NOTES section of the man page.
- There is no requirement on how to handle improper requests. If a client reallocs a stack address, frees an already freed pointer, overruns past the end of an allocated block, or other such incorrect usage, your response can be anything, including crashing or corrupting the heap. We will not test on these cases.
- An allocated block must be at least as large as requested, but it is not required to be exactly that size. The maximum size request your design must accommodate is specified as a constant in allocator.h. If asked for a block larger than the max size, your allocater can return NULL, just as it would for any request that cannot be serviced. You should not assume the value of this constant beyond that it will not be larger than the max value of a size_t.
- Every allocated block must be aligned to an address that is a multiple of the ALIGNMENT constant, also in allocator.h. You may assume that this value is 8, and it is fine for your code to work only with an alignment of 8. However, you should still use the constant instead of hardcoding the value. This alignment applies to payloads that are returned to the client, not to internal heap data such as headers. It's your choice whether small requests are rounded up to some minimum size.
myinit's job is to properly initialize the allocator's state. It should return true if the initialization was successful, or false if the parameters are invalid / the allocator is unable to initialize. For the parameters passed in, you may assume that the heap starting address is a non-NULL value aligned to the ALIGNMENT constant, and that the heap size is a multiple of ALIGNMENT. You should not assume anything else about the parameters, such as that the heap size is large enough for the heap allocator to use.
- Your allocator cannot invoke any memory-management functions. By "memory-management" we specifically mean those operations that allocate or deallocate memory, so no calls to malloc, realloc, free, calloc, sbrk, brk, mmap, or related variants. The use of other library functions is fine, e.g. memmove and memset can be used as they do not allocate or deallocate memory.
- You will need to use a small number of global variables / data, but limited to at most 500 bytes in total. This restriction dictates the bulk of the heap housekeeping must be stored within the heap segment itself.
- You must have an implemented validate_heap function that thoroughly checks the heap data structures and state and returns whether or not any problems were found (see next section).
- You must have an implemented dump_heap function that prints out a representation of what the heap currently looks like (see next section).
validate_heap
- To help identify internal issues, you should implement the validate_heap helper function; it is a function that can check for issues with internal heap allocator state. The test harness calls this function periodically, so if something goes wrong you can be alerted at the exact moment it happens. When implementing validate_heap, augment them by reviewing the internal consistency of the heap. You should write your implementation of validate_heap as you implement each allocator feature - you should not just go back and add it at the end! As your heap data structures become more complex, validate_heap should become more sophisticated to match them.
- dump_heap
- While debugging, it is very useful to be able to see what the layout of the heap (e.g. free blocks, allocated blocks, etc.) looks like at any given point in time. The dump_heap function is a helper function that should do this - it is a function that is not called anywhere in the code, but is useful to call from within GDB (using call) while you are working to print out the current contents of your heap. Like validate_heap, it is an extremely useful tool to gather information, distinguish between a valid and invalid heap configuration and pinpoint the exact moment the heap got out of whack. The bump allocator provides an example dump heap implementation - for your allocators, you should implement your own dump heap function that, for instance, traverses the heap and prints out information about each block (e.g. info in its header, where the next/previous free blocks are in your explicit allocator, etc.), one block per line. This can show you what your heap structure looks like overall, which blocks are allocated vs. free, what size different blocks are, etc.
## 1) Implement An Implicit Free List Allocator
Now you're ready to implement your first heap allocator design. The specific features that your implicit free list allocator must support are:
- Headers that track block information (size, status in-use or free) - you must use the header design mentioned in lecture that is 8 bytes big, using any of the least significant 3 bits to store the status
Free blocks that are recycled and reused for subsequent malloc requests if possible
A malloc implementation that searches the heap for free blocks via an implicit list (i.e. traverses block-by-block).
#### Your implicit free list allocator is not required to:
- implement any coalescing of freed blocks (the explicit allocator will do this)
support in-place realloc (the explicit allocator will do this); for realloc requests you may satisfy the request by moving the memory to another location
- use a footer, as done in the textbook
- resize the heap when out of memory. If your allocator runs out of memory, it should indicate that to the client.
- This allocator won't be that zippy of a performer but recycling freed nodes should certainly improve utilization over the bump allocator.
## 2) Implement An Explicit Free List Allocator
Building on what you learned from writing the implicit version, use the file explicit.c to develop a high-performance allocator that adds an explicit free list as specified in class and in the textbook and includes support for coalescing and in-place realloc.
#### The specific features that your explicit free list allocator must support:
- Block headers and recycling freed nodes as specified in the implicit implementation (you can copy from your implicit version)
- An explicit free list managed as a doubly-linked list, using the first 16 bytes of each free block's payload for next/prev pointers
- Note that the header should not be enlarged to add fields for the pointers. Since the list only consists of free blocks, the economical approach is to store those pointers in the otherwise unused payload.
- Malloc should search the explicit list of free blocks
- A freed block should be coalesced with its neighbor block to the right if it is also free. Coalescing a pair of blocks must operate in O(1) time. You should perform this coalescing when a block is freed.
- Realloc should resize a block in-place whenever possible, e.g. if the client is resizing to a smaller size, or neighboring block(s) to its right are free and can be absorbed. Even if an in-place realloc is not possible, you should still absorb adjacent free blocks as much as possible until you either can realloc in place, or can no longer absorb and must reallocate elsewhere.
#### Your explicit free list allocator is not required to (but may optionally):
- coalesce free blocks to the left or otherwise merge blocks to the left
- coalesce more than once for a free block. In other words, if you free a block and there are multiple free blocks directly to its right, you are only required to coalesce once for the first two. However, for realloc you should support in place realloc which may need to absorb multiple blocks to its right.
- resize the heap when out of memory. If your allocator runs out of memory, it should indicate that to the client.
- This allocator should result in further small gains in utilization and a big boost in speed over the implicit version. You will end up with a pretty snappy allocator that is also strong on utilization 
