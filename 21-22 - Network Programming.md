**Client-server model**: A server <ins>process</ins> and one or more client <ins>processes</ins>. Server provides service for clients by manipulating some resource. A single host/machine can run many servers and clients concurrently.

**Network interface card**: It looks to the computer as an I/O device, similar to a disk controller. Sending a message is writing a message to a virtual file called the _network_. Receiving data is reading the virtual file.

**Network backbone devices**

<ins>Hub</ins>: Connects a collection of hosts to form an _ethernet segment_. A hub slavishly copies each bit from each port to every other port.

<ins>Bridge</ins>: Connects multiple _ethernet segments_. Learns which hosts are reachable from which ports and selectively copy frames from port to port.

<ins>Router</ins>: Connects multiple incompatible LANs physically to form the internet.

**IP (Internet protocol)**: Provides basic naming scheme for hosts and unreliable delivery capability of packets from host-to-host.

**UDP**: Uses IP to provide unreliable datagram delivery from process-to-process.

**TCP**: Uses IP to provide reliable byte streams from process-to-process over connections.

**Socket interface**: Provides the routines required for interprocess communication between applications.

**IP address**: 32-bit IP addresses are stored in IP address struct. Always stored in memory in network byte order (Big Endian). Need to use helper functions to convert to host byte order (Little Endian).

```c
struct in_address {
	uint32_t s_addr; /* network byte order */
}
```

**Helper functions**

```c
// Returns value in network-byte-order
uint32_t htonl(uint32_t hostlong);
uint16_t htons(uint16_t hostshort);

// Returns value in host-byte-order
uint32_t ntohl(uint32_t netlong);
uint16_t ntohs(unit16_t netshort);

// (IP address) "84.52.184.224"
// --> (NBO) 0x 54 34 B8 E0
// --> (HBO) 0x E0 B8 34 54

// Converts dotted-decimal string to binary IP address (NBO)
int inet_pton(AF_INET, const char *src, void *dst);
```

**Internet connection**: Client and servers communicate by sending streams of bytes over connections. Point-to-point, full-duplex and reliable. Connections are uniquely identified by the socket addresses of its endpoints `(cliaddr:cliport, servaddr:servport)`.

<ins>Ephemeral port</ins>: Assigned automatically by client kernel when client makes a connection request.

<ins>Well-known port</ins>: Is associated with some service provided by a server. Mappings between well-known ports and service names are in `/etc/services`.

**Socket**: An `IPaddress:port` pair. To the kernel, a socket is an endpoint of communication. To an application, a socket is a file descriptor for reading/writing from/to the network.

**Socket address structure**

16-byte worth of information. The leading 2 bytes designate what kind of socket it is (TCP, IPv6 etc.).

```c
struct sockaddr {
	uint16_t sa_family; /* Protocol */
	char sa_data[14];   /* Address data */
};
typedef struct sockaddr SA;
```

There is a more specific socket address version for internet called `sockaddr_in`. Must <ins>cast</ins> appropriately as some functions only take in the generic version `sockaddr`.

```c
struct sockaddr_in {
	uint16_t sin_family;         /* Protocol, always AF_INET */
	uint16_t sin_port;           /* Port num in network byte order */
	struct in_addr sin_addr;     /* IP addr in network byte order */
	unsighed char sin_zero[8];   /* Pad to sizeof(struct sockaddr) */
};
```

**Sockets interface overview**

Server runs a program that is ready to receive connections from clients to perform various services -> Start up a client -> Maintain a session, which is back and forth communication between client and server -> Client disconnects -> Server shuts down.

![](images/Pasted%20image%2020220121210749.png)

**Socket descriptor**: Clients and servers use the socket function (System call) to create a generic socket descriptor.

```c
int socket(int domain, int type, int protocol)

// IPv4, stream of bytes (connection endpoint)
int clientfd = Socket(AF_INET, SOCKET_STREAM, 0);
```

**Socket bind**: Asks the kernel to associate the server's socket address with a socket descriptor. The process can read bytes that arrive on the connection whose endpoint is `addr` by reading from the descriptor `sockfd`. Writes to the socket descriptor are transferred along the same connection.

```c
int bind(int sockfd, SA *addr, socklen_t addrlen);
```

**Socket listen**: Converts the active socket (default) to passive socket. Tells the kernel that the descriptor will be used by a server rather than a client. Active socket means the client is responsible for the active opening of TCP connection (SYN). Passive socket means the server waits for requests (SYN) and creates active sockets for each connection accepted.

```c
/* sockfd becomes listenfd */
/* backlog is the number of connection requests in queue */
int listen(int sockfd, int backlog); 
```

**Socket accept**: Server blocks in the function to wait for connection requests from clients on `listenfd`. On connection, the function returns a `connfd` that can be used to communicate with the client via I/O routines. This allows the program to fork a child whenever a new request is received.

```c
int accept(int listenfd, SA *addr, int *addrlen); /* Returns connfd */
```

**Socket connect**: Client establishes a connection with a server at the socket address `addr`. If successful, the `clientfd` is ready for reading and writing. No binding with specific address is necessary as there is an internal bind to an ephemeral port.

```c
int connect(int clientfd, SA *addr, socklen_t addrlen);
```

![](images/Pasted%20image%2020220121215213.png)
![](images/Pasted%20image%2020220121215220.png)

**Socket get address info**

The modern way to convert string representations of host addresses, ports, service names to socket address structures. Reentrant, thus can be used in threaded programs. Convenient for the developer as the API returns in Network Byte Order.

```c
int getaddrinfo(const chat *host,              /* Hostname|address */
                const char *service,           /* Port or service */
				const struct addrinfo *hints,  /* Input parameters */
				struct addrinfo **result);     /* Output linkedlist */
				
void freeaddrinfo(struct addrinfo *result);    /* May ret errcode */

const char *gai_strerror(int errcode);         /* Error message */
```

The result is a linkedlist of `addrinfo` as sometimes a hostname can map to multiple addresses. For example, _google.com_ has more than one IP address.

Clients can walk this list to try each `ai_addr` and `ai_addrlen` until `socket` and `connect` succeed. Servers can walk the list until calls to `socket` and `bind` succeed.

```c
struct addrinfo {
	int ai_flags;             /* Hints argument flags */
	int ai_family;            /* First arg to socket function */
	int ai_socktype;          /* Second arg to socket function */
	int ai_protocol;          /* Third arg to socket function */
	char *ai_canonname;       /* Canonical host name */
	size_t ai_addrlen;        /* Size of ai_addr struct */
	struct sockaddr *ai_addr; /* Ptr to socket address struct */
	struct addrinfo *ai_next; /* Ptr to next addrinfo */
}

// ai_flags
// AI_ADDRCONFIG - Recommended for connections
// AI_NUMERICSERV - Service argument is port number instead of name
// AI_PASSIVE - Returns potential passive sockets, host = NULL
```

<ins>Example of linked list usage</ins>

```c
#include "csapp.h"

int main(int argc, char **argv) {
	struct addrinfo *p, *listp, hints;
	char buf[MAXLINE];
	int rc, flags;

	memset(&hints, 0, sizeof(struct addrinfo));
	hints.ai_family = AF_INET;        /* IPv4 only */
	hints.ai_socktype = SOCK_STREAM;  /* TCP connections only */
	if ((rc = getaddrinfo(argv[1], NULL, &hints, &listp)) != 0) {
		fprintf(stderr, "getaddrinfo error: %s\n", gai_strerror(rc));
		exit(1);
	}
	
	flags = NI_NUMERICHOST; /* Display address instead of name */
	for (p=listp, p; p= p->ai_next) {
		Getnameinfo(p->ai_addr, p->ai_addrlen,
					buf, MAXLINE, NULL, 0, flags);
		printf("%s\n", buf);
	}
	
	Freeaddrinfo(listp);
	exit(0);
}
```

**Socket get name info**: Inverse of `getaddrinfo`, converts socket address to the corresponding host and service. Reentrant and protocol independent.

```c
itn getnameinfo(const SA *sa, socklen_t salen,
                char *host, size_t hostlen,
				char *serv, size_t servlen,
				int flags);

// AI_NUMERICHOST - Return numeric address instead of domain name
// AI_NUMERICSERV - Service argument is port number instead of name
```

**Helper for client to establish connection**

Establish a client connection with a server and return the socket descriptor.

```c
int open_clientfd(char *hostname, char *port) {
	int clientfd;
	struct addrinfo hints, *listp, *p;
	
	/* Get a list of potential server addresses */
	memset(&hints, 0, sizeof(struct addrinfo));
	hints.ai_socktype = SOCK_STREAM; /* TCP connection */
	hints.ai_flags = AI_NUMERICSERV; /* Port is numeric */
	hints.ai_flags |= AI_ADDRCONFIG; /* Recommended for conns */
	Getaddrinfo(hostname, port, &hints, &listp);
	
	/* Walk the list to connect */
	for (p=listp; p; p->ai_next) {
		/* Create socket descriptor */
		if ((clientfd = socket(p->ai_family, p->ai_socktype,
								p->ai_protocol)) < 0)
			continue;
		if (connect(clientfd, p->ai_addr, p->ai_addrlen) != -1)
			break; /* Success */
		Close(clientfd);
	}
	
	/* Clean up */
	Freeaddrinfo(listp);
	if (!p)
		return -1;
	else
		return p;
}
```

**Helper for server to listen**

Create a listening descriptor that can be used to accept connections.

```c
int open_listenfd(char *port) {
	struct addrinfo hints, *listp, *p;
	int listenfd, optval=1;
	
	/* Get a list of potential server addresses */
	memset(&hints, 0, sizeof(struct addrinfo));
	hints.ai_socktype = SOCK_STREAM;
	hints.ai_flags = AI_PASSIVE | AI_ADDRCONFIG;
	hints.ai_flags |= AI_NUMERICSERV;
	Getaddrinfo(NULL, port, &hints, &listp); /* This machine */
	
	/* Walk the list to bind */
	for (p=listp; p; p->ai_next) {
		/* Create socket descriptor */
		if ((listenfd = socket(p->ai_family, p->ai_socktype,
								p->ai_protocol)) < 0)
			continue;
		
		/* Eliminates "Address already in use" error from bind */
		Setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR,
					(const void *)&optval, sizeof(int));
		
		/* Bind the descriptor to the address */
		if (listen(listenfd, p->ai_addr, p->ai_addrlen) == 0)
			break;
		Close(listenfd);
	}
	
	/* Clean up */
	Freeaddrinfo(listp);
	if (!p)
		return -1;
	/* Make it a listening socket to accept connection requests */
	if (listen(listenfd, LISTENQ) < 0) {
		Close(listenfd);
		return -1;
	}
	return listenfd;
}
```

**Echo client**

Sends the stdin to the server until EOF character. Prints out data the server sends back. Similar to CLI application `telnet`.

```c
#include "csapp.h"

int main(int argc, char **argv) {
	int clientfd;
	char *host, *port, buf[MAXLINE];
	rio_t rio;
	
	host = argv[1];
	port = argv[2];
	clientfd = Open_clientfd(host, port);
	Rio_readinitb(&rio, clientfd);
	
	/* Return on new line, NULL on EOF */
	while (Fgets(buf, MAXLINE, stdin) != NULL) {
		Rio_writen(clientfd, buf, strlen(buf));
		
		/* Read server's data into buffer */
		Rio_readlineb(&rio, buf, MAXLINE);
		Fputs(buf, stdout);
	}
	/* Client initiates the termination */
	Close(clientfd);
	exit(0);
}
```

**Iterative echo server**

Reads and echo data from the client connnection until EOF (Client closing). Only supports one connection.

```c
#include "csapp.h"
void echo(int connfd);

int main(int argc, char **argv) {
	int listenfd, connfd;
	socklen_t clientlen;
	struct sockaddr_storage clientaddr; /* Enough for any addrtype */
	char client_hostname[MAXLINE], client_port[MAXLINE];
	
	listenfd = Open_listenfd(argv[1]);
	while (1) {
		clientlen = sizeof(struct sockaddr_storage);
		connfd = Accept(listenfd, (SA *)&clientaddr, &clientlen);
		Getnameinfo((SA *)&clientaddr, clientlen,
				client_hostname, MAXLINE, client_port, MAXLINE, 0);
		printf("Connected to (%s, %s)\n", 
			   client_hostname, client_port);
		echo(connfd);
		Close(connfd);
	}
	exit(0);
}

void echo(int connfd) {
	size_t n;
	char buf[MAXLINE];
	rio_t rio;
	
	Rio_readinitb(&rio, connfd);
	/* Read until the client sends EOF */
	while((n = Rio_readlineb(&rio, buf, MAXLINE)) != 0) {
		printf("server received %d bytes\n", (int)n);
		Rio_writen(connfd, buf, n);
	}
}
```

**Web content**: Sequence of bytes with an associated MIME (Multipurpose Internal Mail Externsion).

**HTTP request**: A request line followed by zero or more request headers.

```
Request line - <method> <uri> <version>
Headers - <header name>: <header data>
```

**HTTP response**: A response line followed by zero or more response headers followed by content.

```
Response line - <version> <status code> <status msg>
Headers - <header name>: <header data>
```

**Tiny web server for static content**

Serves static and dynamic content for `GET` requests. Full code [here](http://csapp.cs.cmu.edu/2e/ics2/code/netp/tiny/tiny.c).

In the example below, the server has already received a particular filename and looked up its size.

```c
void serve_static(int fd, char *filename, int filesize) {
	int srcfd;
	char *srcp, filetype[MAXLINE], buf[MAXBUF];
	
	get_filetype(filename, filetype);
	sprintf(buf, "HTTP/1.0 200 OK\r\n");
	sprintf(buf, "%sServer: Tiny Web Server\r\n", buf);
	sprintf(buf, "%sConnection: close\r\n", buf);
	sprintf(buf, "%sContent-length: %d\r\n", buf, filesize);
	sprintf(buf, "%sContent-type: %s\r\n\r\n", buf, filetype);
	
	/* Response Body */
	srcfd = Open(filename, O_RDONLY, 0);
	srcp = Mmap(0, filesize, PROT_READ, MAP_PRIVATE, srcfd, 0);
	Close(srcfd);
	Rio_writen(fd, srcp, filesize);
	Munmap(srcp, filesize);
}
```

**Issue in serving dynamic content**: The client has to know how to pass the program arguments to the server which then passes to its child process. The server has to capture the content produced by the child process and pass back to the client.

**Common gateway interface**: Defines a standard for transferring information between the client, the server and the child process.

<ins>Example interface for GET</ins>: Arguments list starts with `?` and chains with `&` in the URI. The server creates an environment variable `QUERY_STRING` with all the parameters concatenated. After forking the child process, stdout is redirected to the socket descriptor. The CGI program retrieves arguments from environment variables and outputs to stdout.

**Tiny web server dynamic content**

If the request line URI contains "cgi-bin", dynamic content is served. The server creates a child process and runs the program determined by the URI.

```c
void serve_dynamic(int fd, char *filename, char *cgiargs) {
	char buf[MAXLINE], *emptylist[] = { NULL };
	
	sprintf(buf, "HTTP/1.0 200 OK\r\n");
	Rio_writen(fd, buf, strlen(buf));
	sprintf(buf, "Server: Tiny Web Server\r\n");
	Rio_writen(fd, buf, strlen(buf));
	
	if (Fork() == 0) {
		/* Real server would set all CGI vars here */
		setenv("QUERY_STRING", cgiargs, 1);
		Dup2(fd, STDOUT_FILENO);         /* Redirect stdout to fd */
		Execve(filename, emptylist, environ); /* Runs CGI program */
	}
	Wait(NULL); /* Parent reaps child */
}
```

<ins>Note</ins>: the child process knows the content type and length. So it must generate those response headers.
