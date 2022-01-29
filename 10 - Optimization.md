**General Optimizations**

**Code motion**: Reduce computation frequency.

```c
for (j=0; j<n; j++)
	a[n*i+j] = b[j];

int ni = n*i;
for (j=0; j<n; j++)
	a[ni+j] = b[j];
```

**Reduction in strength**: Replace a costly operation.

```c
for (i=0; i<n; i++) {
	int ni = n*i;
	for (j=0; i<n; j++)
		a[ni+j] = b[j];
}

int ni = 0;
for (i=0; i<n; i++) {
	for (j=0; i<n; j++)
		a[ni+j] = b[j];
	ni += n;
}
```

**Share common expressions**

```
up    = val[(i-1)*n + j];
down  = val[(i+1)*n + j];
left  = val[i*n + j-1];
right = val[i*n + j+1];

long inj = i*n + j;
up    = val[inj - n];
down  = val[inj + n];
left  = val[inj - 1];
right = val[inj + 1];
```

**Optimization blocker**

Programmer must put in effort to write programs in a way that the compiler can transform into efficient machine code.

<ins>Memory aliasing</ins>: In performing safet optimizations, the compiler must assumed that pointers may be aliased. Aliasing is when two different memory references specify single location.

```c
/* twiddle1 CANNOT be optimized to twiddle2 */
/* Will break if yp points to xp */

void twiddle1(long *xp, long *yp) {
	*xp += *yp;
	*xp += *yp;
}
void twiddle2(long *xp, long *yp) {
	*xp += 2* *yp;
}
```

<ins>Procedure calls</ins>: Compiler treats procedure call as a black box as it may have side effects.

```c
/* strlen is an O(N) operation */
/* A better way is to calculate once before the loop */

void lower(char *s) {
	size_t i;
	for (i=0; i<strlen(s); i++) # Compiler will not optimize this
		if (s[i] >= 'A' && s[i] <= 'Z')
			s[i] -= ('A' - 'a');
}
```

**Instruction-level parallelism**

<ins>Superscalar design</ins>: The processor can issue multiple instructions in a single clock. One way is the out-of-order execution where independent instructions are executed in parallel.

<ins>Pipelining</ins>: Divides an instruction into steps where each step is executed in a different part of the processor. The example shows completion of 3 multiplications in 7 cycles, even though each individually requires 3 cycles.

```
long p1 = a * b;
long p2 = a * c;
long p3 = p1 * p2;
```

![](images/Pasted%20image%2020211212175227.png)

**Example of 5-stage pipeline, 2-instructions**

![](images/Pasted%20image%2020220510191420.png)

**SIMD instruction**: Single instruction, multiple data. Floating point multipler can do 8 single-precision floating point multiplications in 3 cycles. (4 double-precision multiplications in 3 cycles)