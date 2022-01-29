**General cache organization**

![](images/Pasted%20image%2020211223102118.png)

**Cache size**: S x E x B bytes. The tag bits and the valid bit are not included in the calculation.

**Cache READ**: When CPU sends address to cache, cache divides the address into a number of regions (tag, set index, block offset).

<ins>Set index</ins>: Locate the set in the cache

<ins>Tag</ins>: Check if any line matches the tag

<ins>Block offset</ins>: Byte location in a cache line

![](images/Pasted%20image%2020211223104905.png)

**Direct-mapped cache simulation**

Direct-mapped cache contains 1 line per set only.
B = 2 bytes/block, S = 4 sets, E = 1 block/set, 4-bit address.

```
// t=1, s=2, b=1
// One-byte per read

0  [0000], miss
1  [0001], hit
7  [0111], miss
8  [1000], miss // Prev line is evicted
0  [0000], miss // Conflict miss
```

<ins>Note</ins>: Individual data can be padded to prevent thrashing via conflict miss.

**E-way associative cache**: (E = 2) means 2-way associative. Searches for matching tag in all the lines.

**2-way associative cache simulation**

B = 2 bytes/block, S = 2 sets, E = 2 blocks/set, 4-bit address.

```
// t=2, s=1, b=1 
// One-byte per read

0  [0000], miss
1  [0001], hit
7  [0111], miss
8  [1000], miss
0  [0000], hit
```

<ins>Line replacement</ins>: LRU and LFU are some examples. At further away from CPU in the memory hierarchy, the higher the cost of a miss. So the overhead of replacement policies are justified.

**Cache WRITE**: Multiple copies of data exist in L1, L2, L3, main memory, disk.

**Write-hit**

<ins>Write-through</ins>: Write immediately to memory on cache hit. Simple but causes bus traffic every write.

<ins>Write-back (Common)</ins>: Defer write to memory until the replacement of line in cache. Another bit is needed to indicate whether the cache block has been modified.

**Write-miss**

<ins>Write-allocate (Common)</ins>: Loads from the lower level into cache and updates the line.

<ins>No-write-allocate</ins>: Writes the word directly to the next lower level.

**Cache hierarchy**

![](images/Pasted%20image%2020211223133236.png)

Separating i-cache and d-cache ensures that data access do not create conflict misses with instructional access. I-caches are typically read-only (simpler).

<ins>L1 i-cache and d-cache </ins>: 32 kB, 8-way associative, 4 cycles.

<ins>L2 unified cache</ins>: 256 kB, 8-way associative, 10 cycles.

<ins>L3 unified cache</ins>: 8 MB, 16-way associative, 40-75 cycles.

**Cache metrics**

<ins>Miss rate</ins>: 3-10% for L1, less than 1% for L2.

<ins>Hit rate</ins>: 1 - miss rate.

<ins>Hit time</ins>: Time required to transfer data to CPU.

<ins>Miss penalty</ins>: 10 cycles for L1 miss, 50 cycles for L3 miss, 200 cycles for main memory miss.

**Cache design considerations**

<ins>Cache size</ins>: Larger cache increases hit rate but increases hit time.
	
<ins>Block size</ins>: Larger block sizes exploit spatial locality but decreases the number of cache lines. This means temporal locality is exploited less.

<ins>Associativity</ins>: Higher associativity reduces conflict misses but increases hit time and cost of implementation.

<ins>Write strategy</ins>: Write-through cache means that read misses are less expensive because they do not trigger memory write (Multi-cache). Write-back caches means fewer data transfers thus allowing more bandwidth for other devices.

**Memory mountain**

Iterate over the first "elem" number of elements (bytes) in an array with stride of "stride", with 4x4 loop unrolling.

As stride increases, spatial locality decreases. As size increases, temporal locality decreases, because there are fewer caches in the hierachy that can hold the data.

![](images/Pasted%20image%2020211223135944.png)

Highest peak of L1 is at 14 GB/s while lowest point of main memory is at 900 MB/s. Even when working set is too large to fit in any caches, the highest point on main memory is 8x higher than lowest point.

<ins>Prefetching</ins>: At stride 1, read throughput is relatively flat because of a hardware prefetching mechanism that identifies stride-1 reference patterns.

**Matrix multiplication example**

Multiple N x N matrix where each element is a double (8 bytes). Block size is 32 bytes (4 doubles) and cache is not big enough to hold multiple rows.

![](images/Pasted%20image%2020211223150051.png)

```c
/* ijk */

for (i=0; i<n; i++) {
	for (j=0; j<n; j++) {
		sum = 0.0;
		for (k=0; k<n; k++)
			sum += a[i][k] * b[k][j]; /* Focus here */
		c[i][j] = sum
	}
}

// Miss per inner loop iteration:
// A: 0.25, B: 1, C: 0

/* kij */

for (k=0; k<n; k++) {
	for (i=0; i<n; i++) {
		r = a[i][k];
		for (j=0; j<n; j++)
			c[i][j] += r * b[k][j]; /* Focus here */
	}
}

// Miss per inner loop iteration:
// A: 0, B: 0.25, C: 0.25

/* jki */

for (j=0; j<n; j++) {
	for (k=0; k<n; k++) {
		r = b[k][j];
		for (i=0; i<n; i++)
			c[i][j] += a[i][k] * r; /* Focus here */
	}
}

// Miss per inner loop iteration:
// A: 1, B: 0, C: 1
```

**Blocking example (Before)**

Blocking improves temporal locality. Assume cache size C << n and cache block is 8 doubles.

```c
c = (double *) calloc(sizeof(double), n*n);

/* Multiply n x n matrices a and b */
void mmm(double *a, double *b, double *c, int n) {
	int i, j, k;
	for (i=0; i<n; i++)
		for (j=0; j<n; j++)
			for (k=0; k<n; k++)
				c[i*n + j] += a[i*n + k] * b[k*n + j];
}
```

Each iteration: n/8 + n = 9n/8

Total misses: 9n/8 \* n<sup>2</sup> = (9/8) \* n<sup>3</sup>

**Blocking example (After)**

Any row or column contains n/B blocks. Block size is 8 doubles. (B<sup>2</sup>)/8 misses for each block.

One block of C: 2n/B \* (B<sup>2</sup>)/8 = nB/4

All blocks of C: nB/4 \* (n/B)<sup>2</sup> = (n<sup>3</sup>)/4B

Ensure that three blocks fit into cache: 3(B<sup>2</sup>) < C.