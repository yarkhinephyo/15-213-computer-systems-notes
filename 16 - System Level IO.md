**Linux file**: Sequence of bytes.
Even I/O devices and sockets are represented as files in Unix.

**Seek**: Changing the current file position. Indicates next offset in the file to read or write.

**File types**
Regular files contain arbitrary data.
Directory is an index for a related group of files.
Socket is for communicating with a process on another machine.
(Out of scope) Pipe, Symbolic link, Character and block devices.

**Text file**
Kernel does not know the difference between text files and binary files. They are differentiated at the application level, which ensures all are ASCII or unicode characters.
EOL in Linux: `\n` (0xa)
EOL in windows: `\r\n` (0xd 0xa)

**Directory**
Consists of an array of links where each link maps a filename to a file. At least two entries which are `.` and `..`

**Opening file**
Informs the kernel that the user is ready to access the file. Takes in a flag which can be bitwise OR. Returns a small identifying integer "file descriptor" where -1 means an error occurred.

```c
int fd;
if ((fd = open("/etc/hosts", O_RDONLY)) < 0) {
	perror("open");
	exit(1);
}
```

**Closing file**
Closing an already closed file is a recipe of disaster in threaded programs. Always check return codes for error.

**Reading file**
Read at least one byte but no more than the number told. Assume that the `read()` call will not reach this number.

For std input, it waits till a string is typed in with `\n`. If it is network connection, it waits till something arrives.

Returns `ssize_t` which is a signed long int. 0 means EOF. +ve is number of bytes. -ve means an error occurred.

```c
char buf[512];
int fd;     /* File descriptor */
int nbytes; /* Number of bytes read */

if ((nbytes = read(fd, buf, sizeof(buf))) < 0) {
	perror("read");
	exit(1);
}
```

**Writing file**
Write at least one byte but no more than the number told. Not all the text provided may be written!

**Short counts**: Reading less than bytes allocated to the buffer.

Encountering EOF on reads.
Reading text lines from a terminal.
Reading and writing network sockets.

**Unbuffered RIO**
Unbuffered input and output of binary data: `rio_readn` and `rio_writen`. Especially useful for transferring data on network sockets.

For reading, short count is returned on EOF, else error is thrown.
For writing, the bytes written will be of the provided size.

```c
/* Robustly read n bytes (unbuffered) */
ssize_t rio_readn(int fd, void *userbuf, size_t n) {
	size_t nleft = n;
	ssize_t nread;
	char *bufp = userbuf;
	while (nleft > 0) {
		if ((nread = read(fd, bufp, nleft)) < 0) {
			if (errno = EINTR) /* Interrupted by sig handler */
				nread = 0;     /* call read() again */
			else
				return -1;     /* errno is set by read() */
		} else if (nread == 0)
			break;             /* EOF */
		nleft -= nread;
		bufp += nread;         /* Move the pointer forward */
	}
	return (n - nleft);
}
```

**Buffered RIO**
Buffered input of text lines and binary data: `rio_readlineb` and `rio_readnb`. Thread-safe.

```c
typedef struct {
	int rio_fd;
	int rio_cnt;      /* Unread bytes in internal buffer */
	char *rio_bufptr; /* Next unread byte */
	char rio_buf[RIO_BUFSIZE];
} rio_t;

/* Associates a file descriptor with a read buffer */
void rio_readinitb(rio_t *rp, int fd);

ssize_t rio_readlineb(rio_t *rp, void *usrbuf, size_t maxlen);

ssize_t rio_readnb(rio_t *rp, void *usrbuf, size_t n);
```

File has associated buffer to hold bytes that have been read from file but not yet read by user code.

![](images/Pasted%20image%2020220102233943.png)

```c
/* Robustly read n bytes (Buffered) */
/* The code for rio_readnb is similar to rio_readn */
/* Instead of read(), it uses rio_read() internally */

/* Uses rp->rio_buf as internal buffer before copying to usrbuf */
static ssize_t rio_read(rio_t *rp, char *usrbuf, size_t n) {
	int cnt;
	while (rp->rio_cnt <= 0) { /* Refill if buf empty */
		rp->rio_cnt = read(rp->rio_fd, rp->rio_buf, sizeof(rp->rio_buf));
		
		if (rp->rio_cnt < 0) {
			if (errno != EINTR) /* Allow sig handler interrupt */
				return -1;
		} else if (rp->rio_cnt == 0)
			return 0;
		else
			rp->rio_bufptr = rp->rio_buf; /* Reset buf ptr */
	}
	
	/* Copy min(n, rp->rio_cnt) bytes from internal buf to user buf */
	cnt = n;
	if (rp->rio_cnt < n)
		cnt = rp->rio_cnt;
	memcpy(usrbuf, rp->rioptr, cnt);
	rp->rio_bufptr += cnt;
	rp->rio_cnt -= cnt;
	return cnt;
}
```

**Copying text from std input to std output**

```c
#include "csapp.h"

int main(int argc, char **argv) {
	int n;
	rio_t rio;
	char buf[MAXLINE];
	
	Rio_readinitb(&rio, STDIN_FILENO);
	while ((n = Rio_readlineb(&rio, buf, MAXLINE)) != 0)
		Rio_writen(STDOUT_FILENO, buf, n);
	exit(0);
}
```

**File metadata**: Maintained by kernel. Accessed by users with `stat` or `fstat` functions. Stored in `struct stat` data structure.

**Accessing file metadata**

```c
int main(int argc, char **argv) {
	struct stat stat;
	char *type, *readok;
	Stat(argv[1], &stat);         /* Filename and pointer to stat*/
	if (S_ISREG(stat.st_mode))    /* Determine file type */
		type = "regular";
	else if (S_ISDIR(stat.st_mode))
		type = "directory";
	else
		type = "other";
	if ((stat.st_mode & S_IRUSR)) /* Check read access */
		readok = "yes";
	else
		readok = "no";
	printf("type: %s, read: %s\n", type, readok);
	exit(0);
}
```

**File descriptor table**: Every process has a file descriptor table that contains pointers to the file records in the Open File table shared globally.
**Open file table**: Table of file records shared globally. Each record contains the last read position and the reference count to check if it is still needed.
**v-node table**: Every file has an associated v-node entry which contains information on the file.

**File sharing**: Two distinct file descriptors sharing the same disk file through two distinct entries in the Open File table.

![](images/Pasted%20image%2020220105152432.png)

**Processes sharing file with fork**
Child process inherits its parent's open files. Each `refcnt` increments by 1. The file positions are shared, so if the parent does a read, the child will have its file position shifted too.

![](images/Pasted%20image%2020220105153322.png)

**I/O redirection**: Copies descriptor table entry `oldfd` to entry `newfd`.

Open file A -> Open file B -> Call `dup2(4,1)`. This copies the entry at index 4 to index 1. Writing file descriptor 1 (Original stdout) will write to disk instead.

![](images/Pasted%20image%2020220105154118.png)

**Standard I/O**: Uses a buffer as Unix I/O calls are expensive around 10,000 clock cycles.

```c
#include <stdio.h>

int main() {
	printf("h");
	printf("i");
	printf("!");
	printf("\n");
	fflush(stdout);
	exit(0);
}

// This only calls Unix I/O one time
// write(1, "hi!\n", 4)
```

**When use Standard I/O**: Working with disk or terminal files
**When use Unix I/O**: Inside signal handlers (Async-signal-safe)
**When use RIO**: Reading and writing network sockets