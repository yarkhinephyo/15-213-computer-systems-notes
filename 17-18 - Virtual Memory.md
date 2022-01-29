**Physical address**: Hardware address in physical memory. Used in simple systems such as embedded microcontrollers.

**Linear address space**: Ordered set of contiguous non-negative integer addresses.

<ins>Virtual address space</ins>: Set of `N = 2^n` virtual addresses. Typically larger than physical address space. The same for all processes running on the system.

<ins>Physical address space</ins>: Set of `M = 2^m` physical addresses. Amount of DRAM present in the system.

**Virtual memory advantages**

1. Efficient usage of memory by using DRAM as a cache for an address space stored on disk.
2. Simplifies memory management for each process by providing uniform address space.
3. Protects the address space of each process from corruption by other processes.

**Virtual memory**: A storage allocation scheme in which secondary memory can be addressed as though it were part of the main memory.

The physical memory (DRAM) is used as a cache. The cache blocks are called <ins>virtual pages</ins>.

There are 3 subsets of virtual pages. _Unallocated_ means that the pages have not been created by the VM system.   _Cached_ and _Uncached_ refers to the virtual pages in and not in the physical memory.

![](images/Pasted%20image%2020220106132621.png)

**DRAM cache organization**: Driven by enormous miss penalty. DRAM 10x slower than SRAM. Disk 10,000x slower than DRAM.

1. Large page size (4kB - 4MB).
2. Fully associative. Only 1 set to prevent conflict misses.
3. Highly sophisticated replacement algorithms.
4. Write-back rather than write-through.

**Memory management unit (MMU)**: Translates virtual address given by CPU to physical address in memory.

**Page table**: Array of page table entries (PTEs) that maps virtual pages to physical pages. Per-process kernel data structure in DRAM.

<ins>Page table entry</ins>: Each virtual page has a PTE. If the valid-bit is set, the virtual page is _cached_ and the address indicates the page in physical memory. If the valid-bit is not set, the address indicates the location on disk.

**Page hit**: Reference to a VM word that is in physical memory.

**Page fault**: The address translation hardware infers that page is not in physical memory and triggers page fault (exception).

1. Transfer of control to the kernel's page fault handler
2. Handler evicts a victim and loads the page from disk into memory
3. Handler updates the page table with physical address
4. The faulting instruction is re-executed.

<ins>Demand paging</ins>: Strategy of waiting until the last moment to swap in a page into physical memory.

**Page fault to disk address**: OS maintains a table of virtual address to the data on disk.

If data is uninitialized, a clean physical page may be mapped. Otherwise, the mapping may be in a page file or swap file or an executable.

**Allocating page**: `malloc` implements `sbrk` which allocates a page table entry to a location in disk, but no swapping occurs yet. Accessing the page will now trigger a page fault instead of a segmentation fault.

**Working set**: Set of active virtual pages that are accessed at any point in time. If `Working set < main memory set` -> no disk traffic after compulsory misses.

**Thrashing**: `SUM(working sets) > main memory` -> performance meltdown when pages are swapped in and out continuously.

**Sharing data among processes**: Map virtual pages in multiple processes to map to the same physical page.

This is how shared libraries are implemented. For example, `libc` only has to map into one physical memory.

**VM as memory management**

<ins>Linking simplified</ins>: Uniform virtual address space means that code, data and heap can start at the same addresses after linking.

<ins>Loading simplified</ins>: Loader `execve` only allocates virtual pages for `.text` and `.data` sections. These sections are loaded into main memory only on demand.

<ins>Memory allocation simplified</ins>: A process can request for contiguous virtual pages even if there is no contiguous physical pages.

**VM as memory protection**

Virtual memory extends PTEs with permission bits. For each virtual memory address in one process, it records `SUP`, `READ`, `WRITE`, `EXEC`. MMU checks the bits on each access.

**Address translation**

`MAP(a) = a'` if data at virtual address `a` is in physical address `a'`. `MAP(a) = ∅` if data at virtual address `a` is not in physical memory.

<ins>Symbols</ins>

| Symbol            | Meaning                                    |
| ----------------- | ------------------------------------------ |
| N = 2<sup>n</sup> | No. of addresses in virtual address space  |
| M = 2<sup>m</sup> | No. of addresses in physical address space |
| P = 2<sup>p</sup> | Page size (bytes)                          |
| TLBI              | TLB index                                  |
| TLBT              | TLB tag                                    |
| VPO               | Virtual page offset                        |
| VPN               | Virtual page number                        |
| PPO               | Physical page offset                       |
| PPN               | Physical page number                       |

<ins>Address translation with page table</ins>

Page table base register contains the physical address of page table for current process. Virtual page number is used as an index of the page table to retrieve physical page number. The virtual page offset is concatenated afterwards to produce the actual physical address.

Note: Page table is in memory too.

![](images/Pasted%20image%2020220107103316.png)

<ins>Page hit</ins>

Even for a hit, there is still memory reference to retrieve PTE with PTE address (VPN added to page table starting address).

![](images/Pasted%20image%2020220107104340.png)

<ins>Page fault</ins>

1. When valid bit is 0 (uncached), MMU throws page fault exception
2. Control is transferred to page fault handler
3. A victim is evicted (Paged to disk if dirty)
4. The desired page is fetched into memory and PTE is updated
5. Handler returns control to the original process which restarts the faulting instruction

![](images/Pasted%20image%2020220107110147.png)

**Translation lookaside buffer (TLB)**

Small set-associative hardware cache in MMU. Maps virtual page numbers to physical page numbers. If PTEs have been cached in L1 instead, they may be evicted by other data references.

![](images/Pasted%20image%2020220107112911.png)

<ins>TLB hit</ins>

Eliminates a memory access as VPN can be used to retrieve PTE inside the on-chip MMU.

![](images/Pasted%20image%2020220107113311.png)

**Multi-level page table**

Level 1 table resides in memory and each PTE points to a page table.

```
Two-level page table with 32-bit address

For each virtual address -
10-bit: 0-1023 idx of level 1 page table
10-bit: 0-1023 idx of level 2 page table
12-bit: 4KB virtual page offset
```

![](images/Pasted%20image%2020220107132719.png)

Number of page tables depends on the architecture. Level 1 and 2 tables cover a huge swath of address space. They should always be in TLB. If the program has reasonable locality, most of the TLB lookups will generally hit in the TLB.

<ins>Memory requirements reduction</ins>: Level 2 page tables do not exist if a PTE in level 1 is null. Since only level 1 page table has to be in the memory all the time, there is less pressure on main memory.

![](images/Pasted%20image%2020220115181245.png)

**Simple memory system example**

- 64-bytes page size - 2<sup>6</sup> addresses
- 14-bit virtual address - 8-bit VPN + 6-bit VPO
- 12-bit physical address - 6-bit PPN + 6-bit PPO
- 4-way associative TLB, 16 entries (4 Sets) - 6-bit TLBT + 2-bit TLBI

_TLB Entry - 6-bit TLBT + 6-bit PPN + 1-bit valid_

![](images/Pasted%20image%2020220115195843.png)

_Page table with 256 entries (2<sup>8</sup> virtual page numbers)_

![](images/Pasted%20image%2020220115195851.png)

_System cache with direct-mapped, 16 sets, 4-byte blocks_

12-bit physical address - 6-bit CT + 4-bit CI + 2-bit CO.

![](images/Pasted%20image%2020220115195900.png)

**Address translation examples**

TLB hit and cache hit with virtual address: 0x03D4

```
13 <--------------------- 0

<- TLBT --> <->
0 0 0 0 1 1 1 1 0 1 0 1 0 0
<---- VPN ----> <-- VPO -->

Check TLB: TLB hit
PPN: 0x0D

<-- CT ---> <- CI > <->
0 0 1 1 0 1 0 1 0 1 0 0
<-- PPN --> <-- PPO -->

Check cache: Cache hit
Byte: 0x36
```

TLB miss and cache miss with virtual address: 0x0020

```
13 <--------------------- 0

<- TLBT --> <->
0 0 0 0 0 0 0 0 1 0 0 0 0 0
<---- VPN ----> <-- VPO -->

Check TLB: TLB miss
Check page table (Valid-bit 1 -> No page fault)
PPN: 0x28

<-- CT ---> <- CI > <->
1 0 1 0 0 0 1 0 0 0 0 0
<-- PPN --> <-- PPO -->

Check cache: Cache miss
Byte: Access memory
```

**Intel core i7 memory**

The unified L2 cache ensures that for programs which utilizes more data than instructions or vice versa, the miss penalty for L1 does not grow too large. The time taken to access L3 is much larger since it is located off the CPU chip.

MMU is also supported by L1 and L2 TLBs. L1 i-TLB's 128 entries > L1 d-TLB's 64 entries. One possible reason that the penalty for missing instructions may be much larger.

**End-to-end core i7 address translation**

VPO = PPO = (L1 cache) CI + CO = 12 bits. While address translation is taking place, L1 can be indexed at the same time. Generally TLB is hit, so PPN bits can be used to check against CT bits in the set afterwards.

![](images/Pasted%20image%2020220115204939.png)

<ins>Level 1-3 page table entries</ins>

Intel supports 48-bit virtual address space and 52-bit physical address space. Linux uses 4KB pages and the page tables are 4KB aligned (2<sup>12</sup> addresses). <ins>So each page table takes up one virtual page.</ins> Only 40 most significant bits of the physical addresses are necessary to be stored as 12 zero-bits can be appended.

```
63  XD: Disable or enable instructions fetched from the PTE

62
..  Unused
52

51
..  Page table physical base address (40 most significant bits)
12

11
..  Unused
9

8
7   PS: Page size either 4KB or 4MB
6
5   A: Reference bit set by MMU on reads and writes
4
3   WT: Write-through or write-back cache policy
2   U/S: User or supervisor access permission
1   R/W: Read-only or read-write access
0   P: Child page table present in memory
```

<ins>Level 4 page table entries</ins>

```
63  XD: Disable or enable instructions fetched from this page

62
..  Unused
52

51
..  Page physical base address (40 most significant bits)
12

11
..  Unused
9

8
7
6   D: Dirty bit set by MMU on writes
5   A: Reference bit set by MMU on reads and writes
4
3   WT: Write-through or write-back cache policy
2   U/S: User or supervisor access permission
1   R/W: Read-only or read-write access
0   P: Child page present in memory
```

**Core i7 page tables**: 2<usp>9</sup> entries per table. Most programs require only one L1 page table entry.

1. L1 page table - 512GB region per entry.
2. L2 page table - 1GB region per entry.
3. L3 page table - 2MB region per entry.
4. L4 page table - 4KB region per entry.

**Virtual address space of a linux process**

User address space starts with 0 (bottom), kernel address space starts at 1 (top). Kernel exists in virtual address space of each process.

The "physical memory" region can be used to access physical memory directly. For example, an offset of 0x1 in the region can access physical memory 0x1.

![](images/Pasted%20image%2020220115223403.png)

**Organization of process virtual memory as collection of "areas"**

An _area_ is a contiguous chunk of allocated virtual memory whose pages are related in some way (Code, heap, data etc). The notion allows the virtual address space to have gaps.

Each process has a data structure called `task_struct` which contains a pointer to `mm_struct`.

`mm_struct` contains `pgd` field, the level 1 page table address. When the process is scheduled, the kernel copies `pgd` entry into CR3 register.

`mm_struct` also contains a pointer to a linked list of `vm_area_struct` which identifies the start/end of each area, their read/write permissions and other flags.

**Page fault exception handling**

MMU triggers a page fault if it does not exist in physical memory. The page fault handler carrys out -

1. Is the virtual address legal? Is it within an area defined by some `vm_area_struct`?
2. Is the access legal? Is a store instruction writing to a read-only segment?
3. If not, the handler evicts a victim and swaps in the page.

**Memory mapping**

Linux initializes the contents of a virtual memory area by associating with disk objects. Areas can be backed by regular file on disk (e.g - executable object file) or anonymous file (e.g - nothing). For anonymous file, first fault allocates a physical page full of 0s.

When the page becomes dirty, it is like any other page. Dirty pages are copied back and forth between memory and a special _swap file_.

**Shared object**

Imagine that a process has already mapped a shared disk object into an area of its virtual memory. If another process tries to map the same object, the kernel can point its page table entry to the physical page that is already cached in DRAM.

**Private object (COW)**

Similarly, two virtual address spaces map to the same region in physical memory. However, the area struct is flagged as private copy-on-write.

1. If an instruction writes to one of the pages, a protection fault is triggered
2. Fault handler creates a new copy in the physical memory and update the page table entry
3. The instruction restarts upon handler returning control

![](images/Pasted%20image%2020220115230730.png)

**Fork function virtual address space**

Creates exact copies of `mm_struct`, `vm_area_struct` and page tables. Then the kernal <ins>flags</ins> each `vm_area_struct` in both processes as private copy-on-write. New pages are created with the COW mechanism only during subsequent writes.

**Execve function virtual address space**

1. Deletes the existing `vm_area_struct`.
2. Maps private (COW) areas by creating new area structs for code, data, bss and stack. The code and data are mapped to the executable object file on disk. The bss area is mapped to an anonymous file whose size is contained the executable object file. The stack and heap are also demand-zero, initially zero length.
3. Maps shared areas such as the standard C library during dynamic linking.
4. Set the program counter to the entry point.

![](images/Pasted%20image%2020220115233259.png)

**User-level memory mapping**

Map `len` bytes starting at `offset` from the file descriptor `fd`, preferably at the address `start`.

```c
void *mmap(void *start, int len, int prot, int flags, int fd, int offset);

// start: Choose 0 for no preferred address.
// prot: Protection flags such as PROT_READ, PROT_WRITE...
// flags: MAP_ANON, MAP_PRIVATE, MAP_SHARED...
```

If a virtual address is read after running mmap, the kernel will trigger page fault and the swapped pages will be the contents of the file.

**Using mmap to copy files example**: Copy a file to stdout without transferring data to user space. The write call reads the bytes which are faulted in by the kernel and writes them to stdout. Avoids a step of buffering before the write command.

```c
#include "csapp.h"

void mmapcopy(int fd, int size) {
	char *bufp;
	bufp = Mmap(NULL, size, PROT_READ, MAP_PRIVATE, fd, 0);
	Write(1, bufp, size); /* STDOUT file descriptor */
	return;
}

int main(int argc, char **argv) {
	struct stat stat;
	int fd;
	if (argc != 2) {
		printf("usage: %s <filename>\n", argv[0]);
		exit(0);
	}

	/* Get file-descriptor */
	fd = Open(argv[1], O_RDONLY, 0);
	Fstat(fd, &stat);
	mmapcopy(fd, stat.st_size);
	exit(0);
}
```

<ins>Theoretical difference from read()</ins>

In traditional file I/O involving read, data is copied from the disk to a kernel buffer, then the kernel buffer is copied into the process's heap space for use. In memory-mapped file I/O, data is copied from the disk straight into the process's address space, into the segment where the file is mapped.