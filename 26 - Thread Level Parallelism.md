**Out-of-order processor**: Processor executes instructions in an ordered governed by the availability of input data and execution units instead of the original order in the program.

In single execution, there is one program counter and queue for operations.

![](images/Pasted%20image%2020220202155658.png)

**Hyperthreading**: Converting physical cores into virtual cores. Assumes that most programs do not make use of all the functional units. Multiple threads can share the functional units in the same core while operating independently.

![](images/Pasted%20image%2020220202155711.png)

**Sum program example**: Use multiple threads to sum a sequence of numbers.

Simplest approach is have threads summing into a global variable protected by a semaphore mutex. (Results) Single thread is very slow and it gets slower as more cores are used. Locking and unlocking are very time-consuming tasks.

```c
/* Thread routine */
void *sum_mutex(void *vargp) {
	long myid = *((long *) vargp);
	long start = myid * nelems_per_thread;
	long end = start + nelems_per_thread;
	long i;
	
	for (i=start; i<end; i++) {
		P(&mutex);
		gsum += i;
		V(&mutex);
	}
	return NULL;
}
```

Another approach is to have threads summing into global array element `psum[i]`. Main thread sums elements of `psum` afterwards. (Results) Orders of magnitude faster than `psum_mutex`.

```c
/* Thread routine */
void *sum_array(void *vargp) {
	long myid = *((long *) vargp);
	long start = myid * nelems_per_thread;
	long end = start + nelems_per_thread;
	long i;
	
	for (i=start; i<end; i++) {
		psum[myid] += i;
	}
	return NULL;
}
```

Another approach is to reduce memory reference by summing into a local variable. When translated into machine code, the local variable should be on a register all the time. (Results) Performance improves further.

```c
/* Thread routine */
void *sum_array(void *vargp) {
	long myid = *((long *) vargp);
	long start = myid * nelems_per_thread;
	long end = start + nelems_per_thread;
	long i, sum = 0;
	
	for (i=start; i<end; i++) {
		sum += i;
	}
	psum[myid] = sum;
	return NULL;
}
```

**Memory consistency**: L1 and L2 caches are separate in multipl cores. With write-back policy only modifying the data on the cache, the data may not be consistent.

![](images/Pasted%20image%2020220202163920.png)

**Snoopy cache**: There are tags in the cache lines that store the states. The tagging happens in both main memory and cache.

```
Invalid - Cannot use value
Shared - Readable copy
Exclusive - Writable copy, available to one thread
```

Thread 1 acquires the exclusive copy of element `a`. Thread 2 acquires the exclusive copy of element `b`. Thread 2 requests for a read copy of `a` in a shared communication medium. Thread 1 supplies value `a` and tag it as "shared".

![](images/Pasted%20image%2020220202164622.png)

![](images/Pasted%20image%2020220202164639.png)

If either wants to write, they will have to request for an exclusive copy again which will disable the shared copies.