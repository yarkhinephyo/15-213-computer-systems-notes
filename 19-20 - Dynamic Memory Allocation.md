**Dynamic memory allocation**: Acquire virtual memory at runtime for data structures whose sizes are known only at runtime. Managed in an area of process virtual memory known as "heap".

**Heap**: Collection of variable sized blocks which are either allocated or free.

**Allocate memory**: `void *malloc(size_t size)`
If successful, returns a pointer to a memory block of `size` bytes. Boundary aligned 8-byte for x86 and 16-byte for x86-64.
**Free memory**: `void free(void *p)`
Where p must come from a previous call to `malloc` or `realloc`.

**Malloc example**

```c
#include <stdio.h>
#include <stdlib.h>

void foo(int n) {
	int i, *p;
	p = (int *) malloc(n * sizeof(int));
	if (p == NULL) {
		perror("malloc");
		exit(0);
	}
	for (i=0; i<n; i++)
		p[i] = i;
		
	free(p);
}
```

**Application constraints**
Can issue any sequence of `malloc` and `free` requests.
`free` request must be to a previously `malloc` block.
**Allocator constraints**
Cannot control the size of allocated blocks.
Must respond immediately to `malloc` requests.
Must allocate blocks from free memory.
Must align blocks to alignment requirements.
Can manipulate and modify only in free memory.

**Allocator's goal**: Given a sequence of `malloc` and `free`, maximize throughput and peak memory utilization. These goals are often conflicting.
**Throughput**: Number of operations per unit time.
**Peak memory utilization**: How efficiently the allocator uses the heap. The sum of all the payloads divided by the total size of the heap.

**Internal fragmentation**: Occurs when memory block assigned to the process is larger than the memory requested.
**External fragmentation**: When there is enough aggregate heap memory but no single free block is large enough.

**Track allocated block size**: Requires additional space for every allocated block. These headers decrease the memory utilization.
(In the picture, each box is 4-byte)

![](images/Pasted%20image%2020220116151659.png)

**Track free blocks**
Method one: Implicit list using length to link all blocks.
Method two: Explicit list among the free blocks with pointers.
Method three: Different free lists for different size classes.
Method four: Blocks sorted by size.

**Implicit list**: Besides block length, will have to mark as free or allocated. Since there is 8-byte or 16-byte alignment, the lower-order 3-bit or 4-bit are "always zero" for each block address. The lowest bit of the length can be used as a flag.

For the example below, assume 4-byte words aligned on 8-byte boundary. The headers are not aligned so that the payloads are aligned. The first allocated block (16/1) is for an 8-byte payload only, as the last 4-byte is a padding. Allocated block of size zero terminates the heap.

![](images/Pasted%20image%2020220116153404.png)

Allocation cost: O(n), for looking through the list. Free cost: O(1). Memory usage depends on the allocation policy.

**First fit**: First free block from the beginning.
**Next fit**: First free block starting from end of previous search.
**Best fit**: Choose best free block with fewest bytes left over.

**Allocation by splitting**: Split the block if the allocated space is smaller than free space.

![](images/Pasted%20image%2020220116155322.png)

```c
addblock(p, 4);

void addblock(ptr p, int len) {
	/* Round up to even blocks: 8-byte align */
	int newsize = ((len + 1) >> 1) << 1;
	int oldsize = *p & -2;
	*p = newsize | 1;
	if (newsize < oldsize)
		*(p+newsize) = oldsize - newsize;
}
```

**Freeing by coalescing**: Join the next/previous blocks if they are free. Freeing the next block is simple. To free the previous block, time complexity would be linear with the current method.

**Boundary tags**: Replicate the block length header at the end of blocks (Called footer). The content is identical. The disadvantage is added internal fragmentation.

![](images/Pasted%20image%2020220116162149.png)

For optimization, the allocated blocks do not need footer tags, only free blocks do (For checking previous block). Use one more bit in the length header to track the allocation status of the previous block. If it is not allocated, that previous block will have footer tag. Coalescing can then be carried out.

**Explicit free list**: Maintain list of free blocks with "next" and "prev" pointers after the block length header. Note that these header, footer and pointers impose a minimum block size.

![](images/Pasted%20image%2020220118221006.png)

Compared to implicit list, allocation is faster as only free blocks are checked. Extra space is needed, but only in the free blocks.

**Allocating explicit free list**: Updates 6 pointers. Split as necessary.

**Freeing explicit free list**
LIFO policy inserts the freed block at the beginning of the free list. Constant time but may cause worse fragmentation.
Address-ordered policy inserts the freed block so that free list blocks are in address order. Decreases fragmentation but requires a search algorithm.

**LIFO policy illustration**
Free block at the front, allocated block at the back.

![](images/Pasted%20image%2020220118225402.png)
![](images/Pasted%20image%2020220118225410.png)

Free blocks at the front and back.

![](images/Pasted%20image%2020220118225910.png)
![](images/Pasted%20image%2020220118225917.png)

**Seglist (Segregated list)**
Given an array of free lists for each size class, allocate a new block (Size N) by searching through free lists for size M > N. If no block is found, request additional memory with `sbrk`. Allocate N bytes from the new memory and place the rest in the free list of the largest size class.

Higher throughput because individual size classes are smaller than the entire list. Better memory utilization because first-fit search of segregated free list approximates a best-fit search of the entire heap.

The free lists are stored at the beginning of the heap.

**Implicit memory management**: Application allocates space but the system automatically frees them. Identify "garbage", area of memory that cannot be referenced anymore (No pointers), and free up.

```c
void foo() {
	int *p = malloc(128);
	return; /* p block is now garbage */
}
```

Need to assume that memory manager can distinguish pointers from non-pointers, pointers point to the start of a block and pointers cannot be hidden.

**Garbage collection**: Automatic reclamation of <ins>heap-allocated</ins> storage.

**Classic GC algorithms**

1. Mark-and-sweep collection
2. Reference counting
3. Copying collection
4. Generational collections

**Memory as a directed graph**

A node (Memory block) is reachable if there is a path from a root node (Has pointers into the heap but outside the heap). Examples include registers, global variables, stack variables. Non-reachable nodes are garbage.

![](images/Pasted%20image%2020220121161939.png)

**Mark and sweep collecting**

<ins>Mark phase</ins>: Use one of the extra mark bits at block headers to mark reachable blocks.

```
// Pseudocode
ptr mark(ptr p) {
	if (!is_ptr(p)) return;
	if (markBitSet(p)) return;
	setMarkBit(p);
	for (i=0; i<length(p); i++) // Check all words in the block
		mark(p[i]);
	return;
}

mark(rootPtr);
```

<ins>Sweep phase</ins>: Scan all blocks and free blocks that are unmarked.

```
// Pseudocode
ptr sweep(ptr p, ptr end) {
	while (p < end) {
		if markBitSet(p)
			clearMarkBit();
		else if (allocateBitSet(p))
			free(p);
		p += length(p);
	}
}
```

The function `is_ptr()` checks if a word is by pointer by checking if it points to an allocated block of memory. However, C pointers can point to the middle of an allocated block. The solution is to use a balanced tree and keep track of the allocated blocks.

**C pointer declarations**

```
Precedence: () [] is higher than * &

int *p        # p is a ptr to int
int *p[13]    # p is an array of ptrs to int
int *(p[13])  # <same>
int **p       # p is a ptr to a ptr to int
int (*p)[13]  # p is a ptr to an array of 13 ints
int *f()      # f is a function returning a ptr to int

int (*(*f())[13])()
# f is a function returning ptr to an array[13] of ptrs to functions returning int
```