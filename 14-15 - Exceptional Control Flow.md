**Control flow**: Sequence of instructions for CPU.

**Altering control flow**: Jumps and branches, Procedure call and return. Such instructions allow programs to reacts to changes in _program state_ such as internal variables. However, programs may also need to react to changes _system state_ which cannot be captured in internal variables.

<ins>Exceptional control flow</ins>: A program reacting to changes in the system state that are not captured in the internal variables.

| Level       | Example                                                                                          |
| ----------- | ------------------------------------------------------------------------------------------------ |
| Hardware    | Events detected by the hardware triggers control transfer to exception handlers                  |
| OS          | Kernel transfer control between processes via context switches                                   |
| Application | A process can send a signal to another process to transfer control                               |
| Application | Program can react to errors by sidestepping the usual track discipline and making nonlocal jumps | 

**Exception**: Abrupt change in the control flow in response to some change in the processor’s state.

![](images/Pasted%20image%2020220101223536.png)

**Exception table**: Every type of exception has a unique number which is indexed in a jump table with the exception handling code.

<ins>Exception number</ins>: A number that indexes into the exception table. They can be assigned by processor designers (Example: Page faults, Divide-by-zero)  or by kernel developers (Example: System calls, I/O signals).

![](images/Pasted%20image%2020220504204043.png)

**Difference from procedure call**

1. The return address pushed onto the stack can be the current instruction or the next.
2. Processor states such as condition codes may also be pushed onto the stack.
3. The items are pushed onto the kernel's stack not the user's stack.

**Types of exception**: Asynchronous exceptions are caused by events external to the processor. Synchronous exception are caused by events that occur as a result of executing an instruction.

<ins>Interrupt (async)</ins>: I/O devices trigger interrupts by signaling the interrupt pin. Always returns to the next instruction.

<ins>Traps (sync)</ins>: Intentional exceptions. For example, `syscall` instruction causes a trap and the exception handler calls the appropriate kernel routine. Always returns to the next instruction.

<ins>Faults</ins>: Unintentional but possibly recoverable. For example, page fault handler loads the page from disk and returns control to the instruction. May return to the current instruction or `abort`.

<ins>Aborts</ins>: Unintentional and unrecoverable. The handler returns control to an `abort` routine.

**System calls**: Each call has a unique ID number. It looks like a function call but it is actually transferring control to the kernel.

**System-level function**: C wrapper functions for system calls. For example, when the user calls `open(filename, options)`, the unique ID is placed in %rax before the `syscall` instruction is executed.

```
<__open>:
...
mov  $0x2, %eax  # open is syscall #2
syscall          # Kernel returns value in %rax. -ve means error
cmp  $0xfffffffffffff001, %rax
...
retq
```

**Process**: Instance of a running program. It provides each program with two key abstractions.

<ins>Logical control flow</ins>: Each program seems to have exclusive use of CPU.

<ins>Private address space</ins>: Each program seems to have exclusive use of main memory.

Each program runs within the _context_ of some process. The context consists of states such as program counter, stack, registers, open file descriptors and code/data in memory.

**Multiprocessing (Single Core)**: Single processor executes multiple processes concurrently.

<ins>Context switch</ins>: Saves the context of the current process -> Restores the context of some previous process -> Pass control to the newly restored process.

![](images/Pasted%20image%2020220101111331.png)

**Concurrent process**: Two processes run concurrently if their logical flows overlap in time.

```
---- time ---->
A  --    --
B    --
C      --  --

Concurrent: A&B, A&C
Sequential: B&C
```

**System call error handling**: On error, Linux system-level functions return -1 and set global variable `errno` to indicate cause. Must always check return status of every system-level function.

```c
if ((pid = fork()) < 0) {
	fprintf(stderr, "fork error: %s\n", strerror(errno));
	exit(0);
}
```

**Error-handling wrapper**: Identical interface as the original function with first letter uppercase.

```c
pid_t Fork(void) {
	pid_t pid;
	if ((pid = fork()) < 0) {
		fprintf(stderr, "fork error: %s\n", strerror(errno));
		exit(0);
	}
	return pid;
}

pid = Fork();
```

**Process states**

<ins>Running</ins>: Either executing or waiting to be executed.

<ins>Stopped</ins>: Execution is suspended until further notice. Runs again when the process receives a `SIGCONT` signal.

<ins>Terminated</ins>: Process is stopped permanently. Three reasons - Receiving a signal whose default action is to terminate, returning from the `main` routine, calling the `exit` function.

**Fork**: Parent creates a new running child process with `fork()`. Returns 0 to the child process, returns child's PID to parent process. Cannot predict the execution order of parent and child.

```c
int main() {
	pid_t pid;
	int x = 1;
	
	pid = Fork();
	if (pid == 0) {
		printf("child: x=%d\n", ++x);
		exit(0);
	}
	
	printf("parent: x=%d\n", --x);
	exit(0);
}
```

<ins>Duplicate but separate address space</ins>: The address space of each process is identical. Any changes made to a variable is not reflected in another process.

<ins>Shared files</ins>: The child process inherits the parent's open files. Since `stdout` file is open in the parent, the child's output is also directed to the screen.

**Modeling fork with process graphs**: Each vertex is the execution of a statement. Each edge can be labeled with the current value of variables.

![](images/Pasted%20image%2020220101122220.png)

**Nested forks in parent**

```c
void fork4() {
	printf("LO\n");
	if (fork() != 0) {
		printf("l1\n");
		if (fork() != 0) {
			printf("L2\n");
		}
	}
	printf("Bye\n");
}
```

![](images/Pasted%20image%2020220101122910.png)

**Reaping child process**

When a process terminates, it still consumes system resources in case parent wants to read its states. ITs called a <ins>zombie</ins> process.

Parent uses `wait` or `waitpid` to reap the zombie process. If parent does not reap, the orphaned child will be reaped by the `init` process (PID 1) when the parent terminates.

Reaping is important in long-running processes such as shells and servers.

**Zombie example**

```c
void fork7() {
	if (fork() == 0) {
		printf("Terminating child, pid=%d\n", getpid());
		exit(0);
	} else {
		printf("Running parent, pid=%d\n", getpid());
		while (1)
			; /* Infinite loop */
	}
}
```

![](images/Pasted%20image%2020220101124107.png)

**Wait**: Parent reaps a child by calling the `wait` function. Parameter is a pointer for the child's status and the return value is the `pid` of the child process that terminated.

```c
void fork9() {
	int child_status;
	
	if (fork() == 0) {
		printf("HC: hello from child\n");
		exit(0);
	} else {
		printf("HP: hello from parent\n");
		wait(&child_status);
		printf("CT: child has terminated\n");
	}
	printf("Bye\n");
}
```

![](images/Pasted%20image%2020220101125014.png)

**Waitpid**: If `pid` argument is -1 the wait set includes all the child processes. This behaviour is the same as `wait`.

<ins>Default behaviour</ins>: Suspends the current process until one child in the wait set terminates.

<ins>WNOHANG</ins>: Returns immediately (with 0) if none of the child processes have terminated yet.

**Multiple wait**: If multiple children has terminated, they will be reaped in arbitrary order. Can use macros such as `WIFEXITED` and `WEXITSTATUS` to get more information about the child process.

```c
void fork10() {
	pid_t pid[N];
	int i, child_status;
	
	for (i=0; i<N; i++)
		if ((pid[i] = fork()) == 0) {
			exit(100+i); /* Child */
		}
	for (i=0; i<N; i++) /* Parent */
		pid_t wpid = wait(&child_status);
		if (WIFEXITED(child_status))
			printf("Child %d terminated with exit status %d\n",
				wpid, WEXITSTATUS(child_status));
		else
			printf("Child %d terminate abnormally\n", wpid);

	/* The only normal termination is if there are no more children */
	if (errno != ECHILD)
		unix_error("waitpid error");
	exit(0);
}
```

**Waitpid error conditions**: If the process has no children, the `waitpid` returns -1 and sets `errno` to `ECHILD`. If the function is interrupted by a signal, it returns -1 and sets `errno` to `EINTR`.

**Loading and running programs with execve**

`int execve(char *filename, char *argv[], char *envp[])`
Filename can be object file or script file. Argument list's first item by convention is the filename. Environment variable list contains "name=value" strings.

Overwrites code, data and stack with the same (1) PID, (2) open files and (3) signal context.

![](images/Pasted%20image%2020220101134205.png)

After the file is loaded, the `execve` calls the start-up code which sets up the stack and passes the control to the main routine.

<ins>Example</ins>: Run `/bin/ls -lt /usr/include` in a child process with the current environment.

```c
/* After creating myargv[] */

if ((pid = Fork()) == 0) {
	if (execve(myargv[0], myargv, environ) < 0) {
		printf("%s: Command not found.\n", myargv[0]);
		exit(1);
	}
}
```

![](images/Pasted%20image%2020220101135022.png)

**Linux process hierachy**

```
          [0]
          |
       init [1]
        /  \    \
       /    \   Login shell
    Daemon   \ 
         Login shell
            /    \
         child   child
```

**Shell**: Application program that runs programs on behalf of the user. It takes in an input string and parse into arguments for `execve` in a child process. If it is a foreground job, a `waitpid` is run to reap the child process.

**Signal**: A small message that notifies a process that an event of some type has occurred in the system. Akin to exceptions and interrupts. Only information in a signal is its ID and the fact that it arrived.

ID | Name | Action | Event
------ | ------ | ------ | ------
2 | SIGINT | Terminate | User typed ctrl-C
9 | SIGKILL | Terminate | Kill program (Can't override/ignore)
11 | SIGSEGV | Terminate | Segmentation violation
14 | SIGALRM | Terminate | Timer signal
17 | SIGCHLD | Ignore | Child terminated, stopped or resumed

<ins>Sending a signal</ins>: Kernel updates state in the context of the destination process.
	
<ins>Receiving a signal</ins>: Destination process is forced by the kernel to react in some way.

**Ways to react a signal**

1. Ignore the signal.
2. Terminate the process.
3. Stop the process until SIGCONT signal.
4. Catch the signal by executing a user-level function called signal handler.

Signal handler is similar to a hardware exception handler responding to an asynchronous interrupt. However, signal handlers are in C code and exception handlers are in the kernel.

**Pending signal**: Only one pending signal of each type is allowed. The subsequent signals of the same type sent to the process are discarded.

</ins>Pending bits</ins>: Maintains a 32-bit int where each bit tracks the delivery and receipt of signals.

**Blocked signal**: Process can block the receipt of certain signals. Can be delivered but will not be received till unblocked.

</ins>Blocked bits</ins>: Maintains a 32-bit int where each bit represent blocked signals.

**Process group**: Each process belongs to one process group.

**Sending signals with /bin/kill**: Program that sends signal to a process or process group.

```
# Send SIGKILL to process 24818
/bin/kill -9 24818

# Send SIGKILL to process group 24817
/bin/kill -9 -24817
```

**Sending signals with keyboard**

<ins>Ctrl-C</ins>: Sends SIGINT (terminate) to every job in foreground process group.

<ins>Ctrl-Z</ins>: Sends SIGTSTP (suspend) to every job in the foreground process group.

**Sending signals with kill()**

If PID is zero, the signal is sent to every process in the process group. If PID is less than zero, the signal is sent to the process group |PID|.

```c
void fork12() {
	pid_t pid[N];
	int i;
	int child_status;
	
	for (i=0; i<N; i++) 
		if ((pid[i] = fork()) == 0) {
			/* child infinite loop */
			while (1)
				;
		}
	
	for (i=0; i<N; i++) {
		printf("Killing process %d\n", pid[i]);
		kill(pid[i], SIGINT);
	}
}
```

**Receiving signals**

Before kernal passes control to process "p" during context switching, its signals are computed.

```c
pnb = pending & ~blocked

if (pnb == 0)
	// Pass control to next instruction
else
	// Choose least nonzero bit and force process to receive the signal
	// Repeat for all nonzero bits
	// Pass control to next instruction
```

**Installing signal handler**

Signal function modifies the default action associated with the receipt of the signal.

```c
// Method signature
#include <signal.h>
typedef void (*sighandler_t)(int);
sighandler_t signal(int _signum_, sighandler_t handler);

// Values for sighandler_t
// SIG_IGN: Ignore
// SIG_DFL: Default action for type signum
// Otherwise, address of the user-level signal handler

// Example

void sigint_handler(int sig) {
	printf("Signal received is %d\n", sig);
}

int main() {
	/* Install signal handler */
	if (signal(SIGINT, sigint_handler) == SIG_ERR)
		unix_error("signal error");
	
	/* Wait for receipt of a signal */
	pause();
	
	return 0;
}
```

Handler for a signal of type T can be interrupted by a signal of another type. The same type of signal is implicitly blocked.

![](images/Pasted%20image%2020220102114944.png)

**Signal handler as concurrent flow**

A signal handler is a separate logical flow (not process) that runs concurrently with the main program. Shared global address space.

```
                            -- time -->
process A: while(1);       |--    --
process A: handler(){...}; |    --
process B:                 |  --    --
```

![](images/Pasted%20image%2020220102114655.png)

**Explicitly blocking signals**

```c
sigset_t mask, prev_mask;

Sigemptyset(&mask);
Sigaddset(&mask, SIGINT);

/* Block SIGINT and save prev blocked set */
Sigprocmask(SIG_BLOCK, &mask, &prev_mask);
...
(Will not be interrupted by SIGINT)
...
/* Restore prev blocked set */
Sigprocmask(SIG_SETMASK, &prev_mask, NULL);
```

**Writing safe handlers**

1. Keep handlers as simple as possible.
2. Call only async-signal-safe functions in the handlers.
3. Save and restore `errno` on entry and exit.
4. Protect accesses to shared data structures by blocking all signals temporarity.
5. Declare global variables as `volatile` to prevent unexpected optimizations. (Read/write from memory directly)
6. Declare global flags as `volatile sig_atomic_t`. (Read/write in one uninterruptible step)

**Async-signal-safety**: A function is either reentrant (All variables on stack frame) or non-interruptible by signals.

For example, suppose a program is in the middle of a call to `printf` and a signal occurs whose handler itself calls `printf`. In this case, the output of the two `printf` statements would be intertwined.

This problem cannot be solved by using synchronization primitives because any attempt would produce immediate deadlock.

**Signals are not queued example**

Signals are not queued. So the receipt of one signal means may be more than one signal of the same type has been sent.

```c
// SIGCHLD means at least one children has terminated

int ccount = 0
void child_handler(int sig) {
	int olderrno = errno;
	pid_t pid;
	while ((pid = wait(NULL)) > 0) {
		ccount--;
		Sio_puts("Handler reaped a child\n");
	}
	if (errno != ECHILD)
		Sio_error("wait error");
	errno = olderrno;
}

void fork14() {
	pid_t pid[N];
	int i;
	ccount = N;
	Signal(SIGCHLD, child_handler);
	
	for (i=0; i<N; i++) {
		if ((pid[i] = Fork()) == 0) {
			Sleep(1);
			exit(0); /* Child exists */
		}
	}
	
	while (ccount > 0) /* Parent spins */
		;
}
```

**Portable signal handling with sigaction**

Different versions of Unix can have different signal handling semantics. This wrapper function can be used to unify the behaviour.

```c
sighandler_t *Signal(int signum, sighandler_t handler) {
	struct sigaction action, old_action;
	
	action.sa_handler = handler;
	sigemptyset(&action.sa_mask); /* Block sigs of type being handled */
	action.sa_flags = SA_RESTART; /* Restart syscalls if possible */
	
	if (sigaction(signum, &action, &old_action) < 0)
		unix_error("Signal error");
		
	return (old_action.sa_handler);
}
```

**Synchronizing flows to avoid races**

If SIGCHLD is not blocked before forking, deletejob() in the child process may run before addjob() in the parent process.

```c
void handler(int sig) {
	int olderrno = errno;
	sigset_t mask_all, prev_all;
	pid_t pid;
	
	Sigfillset(&mask_all);
	while((pid = waitpid(-1, NULL, 0)) > 0) { /* Reap child */
		Sigprocmask(SIG_BLOCK, &mask_all, &prev_all);
		deletejob(pid); /* Delete child from job list */
		Sigprocmask(SIG_SETMASK, &prev_all, NULL);
	}
	if (errno != ECHILD)
		Sio_error("waitpid error");
	errno = olderrno;
}

int main(int argc, char **argv) {
	int pid;
	sigset_t mask_all, mask_one, prev_one;
	
	Sigfillset(&mask_all);
	Sigemptyset(&mask_one);
	Sigaddset(&mask_one, SIGCHLD);
	Signal(SIGCHLD, handler);
	initjobs(); /* Initialize job list */
	
	while(1) {
		Sigprocmask(SIG_BLOCK, &mask_one, &prev_one);  // Blk SIGCHLD
		if ((pid = Fork()) == 0) {                     // Child
			Sigprocmask(SIG_SETMASK, &prev_one, NULL); // Unblock
			Execve("/bin/date", argv, NULL);
		}
		Sigprocmask(SIG_BLOCK, &mask_all, NULL);
		addjob(pid);                            // Add child to list
		Sigprocmask(SIG_SETMASK, &prev_one, NULL);     // Unblock
	}
	exit(0);
}
```

**Waiting for signals with sigsuspend**

Uninterruptible version of the commands below. Atomicity eliminates the possibility that the signal is received after the call to `sigprocmask` and before `pause`.

```c
sigprocmask(SIG_BLOCK, &mask, &prev);
pause();
sigprocmask(SIG_SETMASK, &prev, NULL);
```

While condition is needed to ensure that the execution is suspended if other signals, such as SIGINT, are handled by `pause`.

```c
volatile sig_atomic_t pid;

void sigchld_handler(int s) {
	int olderrno = errno;
	pid = Waitpid(-1, NULL, 0);
	errno = olderrno;
}
void sigint_handler(int s){}

int main() {
	sigset_t mask, prev;
	Signal(SIGCHLD, sigchld_handler);
	Signal(SIGINT, sigint_handler);
	Sigemptyset(&mask);
	Sigaddset(&mask, SIGCHLD);
	
	while (1) {
		Sigprocmask(SIG_BLOCK, &mask, &prev); // Block SIGCHLD
		if (Fork() == 0)
			exit(0);
		
		/* Wait for SIGCHLD to be received */
		pid = 0;
		while (!pid) 
			Sigsuspend(&prev); // Pause for any signal
		
		/* Optionally unblock SIGCHLD */
		Sigprocmask(SIG_SETMASK, &prev, NULL);
		
		/* Do more work */
	}
	exit(0);
}
```