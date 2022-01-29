**Computer Arithmetic**: x ^ 2 >= 0 is true for floats but not always for integers.

Even with overflow, Integer Arithmetic is commutative and associative.
```
300 * 400 * 500 * 600 -> 1640261632
400 * 500 * 600 * 300 -> 1640261632
```

This does not apply to floats which can have some numbers disappearing.
```
(1e20 + -1e20) + 3.14 -> 3.14
1e20 + (-1e20 + 3.14) -> 1e20
```

**Assembly**: Understanding assembly is key to machine-level execution model.

**Memory Systems**: Memory is not unbounded. Memory referencing bugs can be hard to find. For example in C, there is no bound checking on arrays.

```c
typedef struct {
	int a[2];
	double d;
} struct_t;

double fun(int i) {
	volatile struct_t s;
	s.d = 3.14;
	s.a[i] = 1073741824; /* May be out of bounds */
	return s.d;
}

// fun(0) -> 3.14
// fun(1) -> 3.14
// fun(2) -> 3.139999866
// fun(3) -> 2.000000610
// fun(4) -> 3.14
// fun(6) -> Segmentation Fault
```

**Performance Outside Asymptotic Complexity**: Memory access pattern can improve the performance.

```c
void copyij(int src[2048][2048], int dst[2048][2048]) {
	int i,j;
	for (i=0; i<2048; i++)
		for (j=0; j<2048; j++)
			dst[i][j] = src[i][j]
}

// Runtime: 4 ms

void copyij(int src[2048][2048], int dst[2048][2048]) {
	int i,j;
	for (j=0; j<2048; j++)
		for (i=0; i<2048; i++)
			dst[i][j] = src[i][j]
}

// Runtime: 81 ms
```

**Networking and Concurrency**
Concepts of I/O. Programming with socket interface to communicate with any machine.