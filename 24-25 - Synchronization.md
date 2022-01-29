**Shared variable**: Multiple threads reference some instance of `x`.

**Conceptual thread memory model**: Each thread has its own thread context and shares the remaining process context.
**Operational thread memory model**: Register values are truly separate but any thread can read and write the stacks of other threads.

**Global variable**: Virtual memory ensures only one instance is available.
**Local variable**: Each thread stack contains one instance of the variable.
**Local static variable**: Variables inside functions with `static` attribute. The scope is limited to the function but only one instance is stored.

**Example of shared variables**
Just for demonstration, not good practice.

```c
char **ptr;     /* Stored in .data - SHARED */

int main() {
	long i;     /* Stored on stack - NOT SHARED */
	pthread_t tid;
	char *msgs[2] = {
		"Hello from foo",
		"Hello from bar"
	};
	
	ptr = msgs; /* Stored on stack - SHARED */
	for (i=0; i<2; i++)
		Pthread_create(&tid, NULL, thread, (void *)i);
	Pthread_exit(NULL);
}

void *thread(void *vargp) {
	long myid = (long) vargp; /* Stored on stack - NOT SHARED */
	static int cnt = 0;       /* Stored in .data - SHARED */
	printf("[%ld]: %s (cnt=%d)\n", myid, ptr[myid], ++cnt);
	return NULL;
}
```

**Example of improper synchronization**
Both threads increment the same global variable `cnt`. The increment step is broken down into a few operations in assembly (Load register -> Increment -> Store to memory). When these operations for the two threads are interleaved randomly by the OS, the final state of the global variable depends on the scheduling decisions.

```c
/* Global shared variable */
volatile long cnt = 0;

int main(int argc, char **argv) {
	long niters;
	pthread_t tid1, tid2;
	
	niters = atoi(argv[1]);
	Pthread_create(&tid1, NULL, thread, &niters);
	Pthread_create(&tid2, NULL, thread, &niters);
	Pthread_join(tid1, NULL);
	Pthread_join(tid2, NULL);
	
	if (cnt != (2 * niters))
		printf("BOOM! cnt=%ld\n", cnt);
	else
		printf("OK cnt=%ld\n", cnt);
}

void *thread(void *vargp) {
	long i, niters = *((long *) vargp);
	for (i=0; i<niters; i++)
		cnt++;
	return NULL;
}
```

```
thread:
	movq   (%rdi), %rcx
	testq  %rcx, %rcx      # Test if zero
	jle    .L2
	movl   %0, %eax        # Set up 'long i'
.L3:
	movq   cnt(%rip), %rcx # L (Load)
	addq   $1, %rdx        # U (Update)
	movq   %rdx, cnt(%rip) # S (Store)
	addq   %1, %rax
	cmpq   %rcx, %rax
	jne    .L3
.L2
```

As shown below, after the two threads increment one time, the global variable can end up with 1 instead of 2. The problem comes about when a thread is interrupted during the L-U-S sequence.

![](images/Pasted%20image%2020220131164754.png)

**Progress graph**: Depicts the discrete "execution state space" of concurrent threads.
**Trajectory**: Sequence of legal state transitions that describes one possible concurrent execution.

![](images/Pasted%20image%2020220131165244.png)

**Critical section**: Instructions should not be interleaved.
**Unsafe region**: Sets of states where critical sections are interleaved.

![](images/Pasted%20image%2020220131165449.png)

**Semaphore**: Non-negative global integer synchronization variable. Classic solution for synchronization.
**P operation**: If s is non-zero, decrement (Atomically) and return immediately to the calling thread. If s is zero, suspend thread. When the thread is restarted by V eventually, decrement s and return control to the calling thread.
**V operation**: Increments s by one (Atomically). If there are any threads suspended in P, restart any of them.

**Mutual exclusion**: Initialize a semaphore to 1 (Called mutex) to associate with each shared variable. Surround the critical sections with `P(mutex)` and `V(mutex)` operations.

```c
#include <semaphore.h>

int sem_init(sem_t *s, unsigned int val);
int sem_wait(sem_t *s); /* P(s) */
int sem_post(sem_t *s); /* V(s) */
```

Semaphore creates a forbidden region that encloses unsafe region.

![](images/Pasted%20image%2020220131172634.png)

**Producer-consumer problem**
Producer waits for empty slot in buffer, inserts item and notifies consumer. Consumer waits for item, removes item and notifies producer.

For example in event-driven graphical user interface, produce detects mouse movements or keys and place in buffer. Consumer retrieves events from buffer and paints the display.

**Producer-consumer on an n-element buffer**
Requires one mutex (Buffer) and two counting semaphores. The producers wait for `slots` semaphore with `P()`. The consumers announce the available slots with `V()`. The opposite is true for `items` semaphore.

**Creating a shared buffer package**
To allow a shared buffer between threads, where each item is an `int`.

```c
#include "csapp.h"

typedef struct {
	int *buf,    /* Buffer array */
	int n;       /* Max number of slots */
	int front;   /* buf[(front+1)%n] is first item */
	int rear;    /* buf[rear%n] is last item */
	sem_t mutex; /* Protects access to buf */
	sem_t slots; /* Counts available slots */
	sem_t items; /* Counts available items */
}

/* Creates empty, bounded, shared FIFO buffer with n slots */
void sbuf_init(sbuf_t *sp, int n) {
	sp->buf = Calloc(n, sizeof(int));
	sp->n = n;
	sp->front = sp->rear = 0;
	Sem_init(&sp->mutex, 0, 1);
	Sem_init(&sp->slots, 0, n); /* Initially, buf has n empty slots */
	Sem_init(&sp->items, 0, 0); /* Initially, buf has 0 items */
}

void sbuf_deinit(sbuf_t *sp) {
	Free(sp->buf);
}

void sbuf_insert(sbuf_t *sp, int item) {
	P(&sp->slots);
	P(&sp->mutex);
	sp->buf[(++sp->rear)%(sp->n)] = item; /* Insert at rear pointer */
	V(&sp->mutex);
	V(&sp->items);      /* Increment semaphore and notify consumers */
}

void sbuf_remove(sbuf_t *sp) {
	int item;
	P(&sp->items);
	P(&sp->mutex);
	item = sp->buf[(++sp->front)%(sp->n)]; /* Remove item at front */
	V(&sp->mutex);
	V(&sp->slots);
	return item;
}
```

**Readers-writers problem**
Generalization of the mutual exclusion problem. Writes must have exclusive access to the object but unlimited number of readers can access the object.

The implementation below favors readers over writers. There is another variant favoring the writers.

The semaphore `mutex` protects the `readcnt`. The semaphore `w` protects the critical reading/writing section.

```c
int readcnt;    /* Initially 0 */
sem_t mutex, w; /* Initially 1 */

void reader(void) {
	while (1) {
		P(&mutex);
		readcnt++;
		if (readcnt == 1) /* First reader in */
			P(&w);
		V(&mutex);
		
		/* Critical reading section */
		
		P(&mutex);
		readcnt--;
		if (readcnt == 0) /* Last reader out */
			V(&w);
		V(&mutex);
	}
}

void writer(void) {
	while (1) {
		P(&w);
		/* Critical writing section*/
		V(&w);
	}
}
```

**Prethreaded concurrent echo server**
Uses producer-consumer model and the shared buffer to implement a concurrent web server that makes use of a pool of worker threads. This removes the overhead of producing and terminating threads.

![](images/Pasted%20image%2020220201224258.png)

The main thread accepts new connections and immediately add to the shared buffer.

```c
sbuf_t sbuf; /* Shared buffer of connected descriptors */

int main(int argc, char **argv) {
	int i, listenfd, connfd;
	socklen_t clientlen;
	struct sockaddr_storage clientaddr;
	pthread_t tid;
	
	listenfd = Open_listenfd(argv[1]);
	sbuf_init(&sbuf, SBUFSIZE);
	for (i=0; i<NTHREADS; i++)
		Pthread_create(&tid, NULL, thread, NULL);
	while (1) {
		clientlen = sizeof(struct sockaddr_storage);
		connfd = Accept(listenfd, (SA *) &clientaddr, &clientlen);
		sbuf_insert(&sbuf, connfd);
	}
}
```

The threads detach and service the client.

```c
void *thread(void *vargp) {
	Pthread_detach(pthread_self());
	while (1) {
		int connfd = sbuf_remove(&sbuf); /* Remove connfd from buf */
		echo_cnt(connfd);
		Close(connfd);
	}
}
```

Before using `echo_cnt`, initialize the `byte_cnt` that adds all the bytes of data from different clients.

```c
static int byte_cnt;
static sem_t mutex; /* Protects byte_cnt */

static void init_echo_cnt(void) {
	Sem_init(&mutex, 0, 1);
	byte_cnt = 0;
}
```

Within the service routine, call the initialization routine only once through the static variable `pthread_once_t`.

```c
void echo_cnt(int connfd) {
	int n;
	char buf[MAXLINE];
	rio_t rio;
	static pthread_once_t once = PTHREAD_ONCE_INIT;
	
	/* Only one thread calls this function */
	Pthread_once(&once, init_echo_cnt);
	Rio_readinitb(&rio, connfd);
	while((n = Rio_readlineb(&rio, buf, MAXLINE)) != 0) {
		P(&mutex);
		byte_cnt += n;
		printf("thread %d received %d (%d total) bytes on fd %d\n",
			  (int) pthread_self(), n, byte_cnt, connfd);
		V(&mutex);
		Rio_writen(connfd, buf, n);
	}
}
```

**Thread safe**: A function always produce correct results when called repeatedly from multiple concurrent threads.

**Thread-unsafe functions**

**Class 1: Do not protect shared variables.**
Fix is to use `P`, `V` semaphore operations. Synchronization will slow down code.

**Class 2: Keep state across multiple invocations.**
Libc `rand` function relies on static state. If multiple threads call the `rand` function, pseudo-random property will be broken. Since other threads jump in to modify `next`, the sequence of numbers will not be the same anymore.

```c
static unsigned int next = 1; 
int rand(void) {
	next = next * 1103515245 + 12345;
	return (unsigned int)(next / 65536) % 32768;
}
/* srand: set seed for rand() */
void srand(unsigned int seed) {
	next = seed;
}
```

Fix is for each thread to store `next` on a stack (After modifying the function).

**Class 3: Return a pointer to a static variable.**
Functions return the same address of a global/static variable. For example, `ctime()` always return the same char pointer. If another thread calls `ctime()`, the content at the char pointer will be modified.

Fix is to lock-and-copy. The caller must free memory.

```c
/* lock-and-copy version */
char *ctime_ts(const time_t *time_p, char *privatep) {
	char *sharedp;
	P(&mutex);
	sharedp = ctime(timep);
	strcpy(privatep, sharedp);
	V(&mutex);
	return privatep;
}
```

**Class 4: Call thread-unsafe functions.**
Fix is to not call them.

**Reentrant function**
No access to shared variables. One way to make Class 2 function thread-safe is to make it reentrant. Efficient way of synchronization.

![](images/Pasted%20image%2020220202000632.png)

**Thread-safe library functions**: All functions in standard C library. Most Unix system calls except the ones below.

Function | Class | Reentrant
------ | ------- | ------ 
asctime | 3 | asctime_r
ctime | 3 | ctime_r
gethostbyaddr | 3 | gethostbyaddr_r
gethostbyname | 3 | gethostbyname_r
inet_ntoa | 3 | (none)
localtime | 3 | localtime_r
rand | 2 | rand_r

**Races**: Correctness of a program depends on one thread reaching point x before another thread reaches point y.

```c
/* Since i address is shared, race condition is created */
/* If dereferencing occurs after i is incremented, its wrong */
int main() {
	pthread_t tid[N];
	int i;
	
	for (i=0; i<N; i++)
		Pthread_create(&tid[i], NULL, thread, &i);
	for (i=0; i<N; i++)
		Pthread_join(&tid[i], NULL);
	exit(0);
}

void *thread(void *vargp) {
	int myid = *((int *) vargp);
	printf("Hello from thread %d\n", myid);
	return NULL;
}
```

There are more duplicated IDs in multi-core servers.

![](images/Pasted%20image%2020220202092048.png)

**Race elimination**: Avoid unintended sharing of state. Memory can be allocated for each pointer being passed into thread.

```c
int main() {
	pthread_t tid[N];
	int i, *ptr;
	
	for (i=0; i<N; i++)
		ptr = Malloc(sizeof(int));
		*ptr = i;
		Pthread_create(&tid[i], NULL, thread, ptr);
	for (i=0; i<N; i++)
		Pthread_join(&tid[i], NULL);
	exit(0);
}

void *thread(void *vargp) {
	int myid = *((int *) vargp);
	Free(vargp);
	printf("Hello from thread %d\n", myid);
	return NULL;
}
```

**Deadlock**: Wait for condition that will never be true.

For example, thread 0 and 1 require resource s0 and s1 to proceed. Thread 0 acquires s0 and waits for s1. Thread 1 acquires s1 and waits for s0.

![](images/Pasted%20image%2020220202170011.png)

![](images/Pasted%20image%2020220202093032.png)