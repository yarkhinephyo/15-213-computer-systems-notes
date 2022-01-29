**Races**: Outcome depends on arbitrary scheduling decisions in the system.
**Deadlock**: Improper resource allocation prevents forward progress.
**Livelock/starvation/fairness**: External events or scheduling decisions prevent sub-task progress.

**Problem with iterative server**
Client 2 returns on `connect` call even though the connection is not accepted because server-side TCP manager queues the request. Client 2 returns after `write` call because server-side TCP manager buffers the input data. Only client 2's `read` call is blocked.

![](images/Pasted%20image%2020220130110032.png)

**Approaches to concurrent servers**

Type | Interleaving by | Address space | Note
------ | ------- | ------ | -------
Process-based | Kernel | Separate | 
Event-based | Programmer | Shared | I/O multiplexing
Thread-based | Kernel | Shared | 

**Process-based concurrent server**
Fork a child request for every client connection so that a blocking client cannot influence the other clients.

Both parent and child have copies of `listenfd` and `connfd` so appropriate connections must be closed. The (long running) server process must reap its children to prevent memory leak.

There is an additional overhead for process control. Furthermore, it is nontrivial to share data between processes.

```c
int main(int argc, char **argv) {
	int listenfd, connfd;
	socklen_t clientlen;
	struct sockaddr_storage clientaddr; /* Handle both IPv4 IPv6 */
	
	Signal(SIGCHLD, sigchld_handler);
	listenfd = Open_listenfd(argv[1]);
	while(1) {
		clientlen = sizeof(struct sockaddr_storage);
		connfd = Accept(listenfd, (SA *) &clientaddr, &clientlen);
		if (Fork() == 0) {
			Close(listenfd); /* Child stops listening */
			echo(connfd);    /* Child services client */
			Close(connfd);   /* Child closes connection with client */
			exit(0);
		}
		Close(connfd);       /* Parent closes connected socket */
	}
}

// Avoid memory leak by reaping children
void sigchld_handler(int sig) {
	while(waitpid(-1, 0, WNOHANG) > 0)
		;
	return;
}
```

**Event-based concurrent server**
Server maintains a set of active connections with an array of `connfd`. It determines which descriptors have pending inputs with `select` or `epoll` functions. If `listenfd` has input, the server accepts connection and add to the `connfd` array.

No process or thread control overhead. Easy to debug due to one logical control flow.

More complex to code then process or thread based designs. Cannot take advantage of multi-core.

![](images/Pasted%20image%2020220130113240.png)

**Process overview**
Program context and kernel context stored in the registers and the private virtual address space (Kernel virtual memory + Process virtual memory).

![](images/Pasted%20image%2020220130115028.png)

**Thread overview**
Independent stack, stack pointer and the thread context.

Each thread has its own logical control flow and stack (Local variables). All thread share the same code, data and kernel context. Context switching threads has a very low overhead.

Threads associated with a process form a pool of peers without hierarchy.

![](images/Pasted%20image%2020220130115314.png)

**Posix threads interface**: Standard interface for manipulation threads in C programs.

**Pthreads hello world**
Creates a peer thread with `pthread_create` and waits for it to finish with `pthread_join`. The `thread` callback takes in one generic pointer.

```c
#include "csapp.h"
void *thread(void *vargp); /* Takes in one generic pointer */

int main() {
	pthread_t tid;
	/* Last argument is a pointer for thread function */
	Pthread_create(&tid, NULL, thread, NULL);
	Pthread_join(tid, NULL);
	exit(0);
}

void *thread(void *vargp) {
	printf("Hello world!\n");
	return NULL;
}
```

**Thread-based concurrent server**
It is important to run detached threads to avoid memory leak. Joinable threads (default configuration) must be reaped with `pthread_join` to free memory resources.

The file descriptor pointer has to be malloc-ed instead of being passed directly. If not, we will be assuming wrongly that the peer thread would dereference the pointer before the main thread gets a new connected file descriptor. By using malloc, the pointer is provided with new memory location every loop.

```c
int main(int argc, char **argv) {
	int listenfd, *connfd;
	socklen_t clientlen;
	struct sockaddr_storage clientaddr; /* Handle both IPv4 IPv6 */
	pthread_t tid;
	
	listenfd = Open_listenfd(argv[1]);
	while(1) {
		clientlen = sizeof(struct sockaddr_storage);
		/* Needed to prevent a race condition */
		/* Now a different pointer is allocated each loop */
		connfdp = Malloc(sizeof(int));
		*connfd = Accept(listenfd, (SA *) &clientaddr, &clientlen);
		Pthread_create(&tid, NULL, thread, connfdp);
	}
}

void *thread(void *vargp) {
	int connfd = *((int *) vargp);
	/* When detached, kernel automatically reaps terminated thread */
	Pthread_detach(pthread_self());
	/* Free data in heap as it has been copied onto stack */
	Free(vargp);
	echo(connfd);
	Close(connfd);
	return NULL;
}
```