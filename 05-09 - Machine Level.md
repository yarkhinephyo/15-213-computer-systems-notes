**x86: Intel processor line**

| Name       | Description                                          |
| ---------- | ---------------------------------------------------- |
| 8086       | First 16-bit Intel processor with 1 MB address space |
| i386       | First 32-bit Intel processor (IA32), Can run Unix    |
| Pentium 4E | First 64-bit processor (x86-64), Hyperthreading      |
| Core 2     | First multi-core Intel processor, No hyperthreading  |
| Core i7    | Four cores, Hyperthreading                           | 

**Assembly Code View**

<ins>Program Counter</ins>: Address of next instruction.

<ins>Register File</ins>: Heavily used program data.

<ins>Condition Code</ins>: Status information about the most recently executed instruction.

<ins>Memory</ins>: Virtual address.

Assembly code makes no distinction between signed, unsigned integers or even pointers.

**C to Assembly**: `gcc -Og -S sum.c`

![](images/Pasted%20image%2020211127174426.png)

The program executed by the machine is simply a sequence of bytes encoding a series of instructions.

**Assembly code suffix**

| Data type        | Suffix | Bytes |
| ---------------- | ------ | ----- |
| Byte             | b      | 1     |
| Word             | w      | 2     |
| Double word      | l      | 4     |
| Quad word        | q      | 8     |
| Single precision | s      | 4     |
| Double precision | l      | 8     |

Note that `l` is used for double word and double precision, but there is no ambiguation since floating-point operations use different set of instructions.

Note that 32-bit instructions will zero the top 32 bits of 64-bit registers while 8-bit and 16-bit instructions do not.

```
movabsq $0xffffffffffffffff, %rax

movb $0, %al  # rax = 0xffffffffffffff00
movw $0, %ax  # rax = 0xffffffffffff0000
movl $0, %eax # rax = 0x0000000000000000
```

**x86-64 Integer Registers**

| Register | Meaning                  | 
| -------- | ------------------------ |
| %exx     | 32-bit                   |
| %rxx     | 64-bit                   |
| %rsp     | Stack register           |
| %rdi     | First argument register  |
| %rsi     | Second argument register |
| %rdx     | Third argument register  |
| %rip     | Instruction pointer      |

**Memory Addressing Mode**

<ins>Normal</ins>: (R)

<ins>Displacement</ins>: D (R) # Mem[Reg[R] + D]

<ins>Indexed</ins>: D (Rb,Ri) # Mem[Reg[Rb] + Reg[Ri] + D]

<ins>Scaled indexed</ins>: D (Rb,Ri,S) # Mem[Reg[Rb] + S\*Reg[Ri] + D] 

```
%rdx 0xf000
%rcx 0x0100

0x8 (%rds)      ->  0xf000 + 0x8     ->  0xf008
(%rdx, %rcx)    ->  0xf000 + 0x100   ->  0xf100
(%rdx, %rcx, 4) ->  0xf000 + 4*0x100 ->  0xf400
0x80(, %rdx, 2) ->  2*0xf000 + 0x80  ->  0x1e080
```

**Moving Data**: `movq Source, Dest`

Immediate: $0x400, Register: %rax, Memory: (%rax). Memory-memory transfer is not allowed.

Immediate source operands are limited to 32-bit two's complement numbers. The `movabsq` can have 64-bit immediate value as the source operand and can only have a register as the destination.

Instructions in `movz` class fill out the remaining bytes of the destination with zeros. Instructions in `movs` class fills out with sign extension.

<ins>Casting example</ins>

```
*dp = (dest_t) *sp;

// unsigned char to long
movzbl (%rdi), %eax  // Zero-extension because unsigned
movq    %rax, (%rsi) // Store 8 bytes

// int to char
movl   (%rdi), %eax  // Read 4 bytes
movb   %al, (%rsi)   // Store low-order byte
```


**Load Effective Address**: `leaq Src, Dest`

1. It can compute the address without a memory reference.
2. It can compute arithmetic expressions of the form x + k * y.

In `movq` (and all other instruction except `leaq`), these parentheses denote a genuine dereference, whereas in `leaq` they do not and are purely convenient syntax.

```
x * 12 = ?

leaq (%rdi, %rdi, 2), %rax #  3 * x
salq $2, %rax              # (3 * x) << 2
```

**Arithmetic Operations**

| Format | Computation                  |
| ------ | ---------------------------- |
| addq   | Dest = Dest + Src            |
| subq   | Dest = Dest - Src            |
| imulq  | Dest = Dest * Src            |
| salq   | Dest = Dest << Src           |
| sarq   | Dest = Dest >> Src           |
| shrq   | Dest = Dest >> Src (Logical) |
| xorq   | Dest = Dest ^ Src            |
| andq   | Dest = Dest & Src            |
| orq    | Dest = Dest \| Src           |

**One Operand Instructions**

| Format | Computation     |
| ------ | --------------- |
| incq   | Dest = Dest + 1 |
| decq   | Dest = Dest - 1 |
| negq   | Dest = - Dest   |
| notq   | Dest = ~Dest    |

**Example Arithmetic Expressions**

![](images/Pasted%20image%2020211127190654.png)

**Condition Code registers**

<ins>CF</ins>: Carry flag. Overflow for unsigned.

<ins>ZF</ins>: Zero flag. Operation yielded zero.

<ins>SF</ins>: Sign flag. Operation yielded negative value.

<ins>OF</ins>: Overflow flag. Overflow for two's complement.

**Explicit set with cmp**: `cmpq src, dest`

```
cmpq b, a is like a - b without a destination

CF set if unsigned overflow
ZF set if a - b == 0
SF set if a - b < 0
OF set if two complement overflow (a > 0 && b < 0 && (a-b) < 0) || ...
```

**Explicit set with test**: `testq src1, src2`

```
testq b, a is like a & b without a destination

ZF set if a & b == 0
SF set if a & b < 0
```

**SetX**: Set low-order byte destination to 0 or 1 based on combinations of condition codes. Does <ins>not</ins> affect other 7 bytes.

Note how (SF ^ OF) is manipulated for signed comparisons.

| Format | Computation      | Description               |
| ------ | ---------------- | ------------------------- |
| sete   | ZF               | Equal / Zero              |
| setne  | ~ZF              | Not equal / Not Zero      | 
| sets   | SF               | Negative                  |
| setns  | ~SF              | Nonnegative               |
| setg   | ~(SF ^ OF) & ~ZF | Greater (Signed)          |
| setge  | ~(SF ^ OF)       | Greater or equal (Signed) |
| setl   | (SF ^ OF)        | Less                      |
| setle  | (SF ^ OF) \| ZF  | Less or equal (Signed)    |
| seta   | ~CF & ~ZF        | Above (Unsigned)          |
| setb   | CF               | Below (Unsigned)          |

<ins>Condition example</ins>

```
int gt (long x, long y) {
	return x > y;
}

# rdi -> x
# rsi -> y
# rax -> return val

gt:
	cmpq    %rsi, %rdi   # Compare x:y
	setg    %al          # Set when > (Can only one byte)
	movzbl  %al, %eax    # Zero other bytes
	ret

# When result is 32-bit result, zeros are added for the other 32-bits
```

**jX**: Jump instructions

| Format | Computation      | Description               |
| ------ | ---------------- | ------------------------- |
| jmp    | 1                | Unconditional             |
| je     | ZF               | Equal / Zero              |
| jne    | ~ZF              | Not equal / Not Zero      |
| js     | SF               | Negative                  |
| jns    | ~SF              | Nonnegative               |
| jg     | ~(SF ^ OF) & ~ZF | Greater (Signed)          |
| jge    | ~(SF ^ OF)       | Greater or equal (Signed) |
| jl     | (SF ^ OF)        | Less                      |
| jle    | (SF ^ OF) \| ZF  | Less or equal (Signed)    |
| ja     | ~CF & ~ZF        | Above (Unsigned)          |
| jb     | CF               | Below (Unsigned)          |

<ins>PC-relative addressing</ins>: Jump instruction commonly use this addressing, which allows encodings to take less bytes and be functional even when the code is shifted around memory.

For example, the jump instruction at 0x3 jumps to 0x8. The relative address 0x3 is added to %rip at 0x5.

```
0x0: 48 89 f8        mov %rdi,%rax
0x3: eb 03           jmp 0x8
0x5: 48 d1 f8        sar %rax
0x8: 48 85 c0        test %rax,%rax
```

<ins>Condition Branch Example</ins>

```
int absdiff (long x, long y) {
	long result;
	if (x > y)
		result = x - y;
	else
		result = y - x;
	return result
}

absdiff:
	cmpq   %rsi, %rdi   # Compare x:y
	jle    .L4          # Jump when <=
	movq   %rdi, %rax
	subq   %rsi, %rax
	ret
.L4:
	movq   %rsi, %rax
	subq   %rdi, %rax
	ret
```

**Conditional Moves**: Compute both if and else statements first. Only after that the choosing is done with the condition. Because branches are very disruptive to instruction flow (Interferes with pipelining).

Not good for complex computations, risky computations, computations with side effects.

```
val = Test ? Then_exp : Else_exp;

# Branching Logic

  ntest = !Test;
  if (nTest) goto Else;
  val = Then_exp;
  goto Done;
Else:
  val = Else_exp;
Done:
  ...

# Conditonal Move Logic

result = Then_exp;
eval = Else_exp;
nt = !Test;

if (nt) result = eval;
return result;
```

<ins>Example</ins>

```
absdiff:
	movq   %rdi, %rax   # x
	subq   %rsi, %rax   # result = x - y
	movq   %rsi, %rdx
	subq   %rdi, %rdx   # eval = y - x
	cmpq   %rsi, %rdi   # Compare x:y
	cmovle %rdx, %rax   # if <=, result = eval
	ret
```

**While loop**

<ins>Jump-to-middle" translation</ins>

```
while (Test)
	Body

# Goto version logic -Og
	
	goto Test;
loop:
	Body
test:
	if (Test)
		goto loop;
done:
```

<ins>Guarded-do translation</ins>

```
# Goto version logic -O1

	if (!Test)
		goto done;
loop:
	Body
	if (Test)
		goto loop;
done:
```

**For loop**: Translate for-loop into while-loop.

```
for (Init; Test; Update)
	Body
	
Init;
while (Test) {
	Body;
	Update;
}
```

**Switch statement**: 

The asterisk `*` means an <ins>indirect jump</ins>. Jump to the value at the address, and not the direct address.

![](images/Pasted%20image%2020211128235325.png)

```
long switch_eg(long x, long y, long z) {
	long w = 1;
	switch (x) {
		...
		case 2:
			w = y/z;
		case 3:
			w += z;
			break;
		...
		case 6:
			w -= x;
			break;
		default;
			w = 2;
	}
	return w;
}


switch_eg:
	movq   %rdx, %rcx
	cmpq   $6, %rdi      # Compare x to 6
	ja     .L8           # Jump default if x < 0 or x > 6
	jmp    *.L4(,%rdi,8) # goto *JTab[x]
	

# Jump Table

.section    .rodata
  .align 8           # Spaced by 8 bytes
.L4
  .quad     .L8
  .quad     .L3
  .quad     .L5      # Case 2
  .quad     .L9      # Case 3  
  .quad     .L8
  .quad     .L7
  .quad     .L7
  
# Code Blocks (x == 2, x == 3)

.L5                    # Case 2
	movq   %rsi, %rax
	cqto
	idivq  %rcx        # y/z
	jmp    .L6
.L9                    # Case 3
	movl   $1, %eax    # w = 1
.L6
	addq   %rcx, %rax  # w += z
	ret
```

**Mechanisms in procedures**

1. Passing Control: Program counter must change accordingly.
2. Passing Data: One or more parameters and one return value.
3. Memory Management: Allocate and free local variables.

**x86-64 stack**: Register %rsp contains the lowest stack address.

**Push onto stack**:  `pushq src`

Fetch operand at src -> decrement %rsp by 8 -> write operand at the new %rsp address.

**Pop from stack**: `popq dest`

Read value from %rsp address -> increment %rsp by 8 -> store the value at dest.

**Call procedure**: `callq <Label>`

Invokes procedure by pushing the return address onto the stack and setting %rip to the starting address of the procedure.

During `ret`, the return address is popped and the %rip is set to this address.

![](images/Pasted%20image%2020211204205245.png)

![](images/Pasted%20image%2020211204205259.png)

![](images/Pasted%20image%2020211204205447.png)

**Stack frames**: Frames for every procedure that has been called but not returned.

```
long incr(long *p, long val) {
	long x = *p;
	long y = x + val;
	*p = y;
	return x;
}

incr:
	movq  (%rdi), %rax
	addq  %rax, %rsi
	movq  %rsi, (%rdi)
	ret

----
long call_incr() {
	long v1 = 15213;
	long v2 = incr(&v1, 3000);
	return v1+v2;
}

call_incr:
	subq  $16, %rsp       # Allocate 16 bytes on the stack
	movq  $15213, 8(%rsp) # Store 15213 at offset 8 from stack ptr
	movl  $3000, %esi
	leaq  8(%rsp), %rdi
	call  incr
	addq  8(%rsp), %rax
	addq  $16, %rsp       # Move back the stack pointer
	ret
```

**Passing parameters**

There are 6 registers in the order of %rdi, %rsi, %rdx, %rcx, %r8, %r9.

If there are more than 6 arguments, the calling procedure puts arguments 7 and above onto the top of the stack. All data sizes are rounded up to be multiples of 8. Considering that the return address is at the top of the stack, the argument 7 is at 8(%rsp), argument 8 is at 16(%rsp) and so on.

**Register saving conventions**

<ins>Caller saved</ins>: Caller saves the temporary values before the call.

<ins>Callee saved</ins>: Callee saves the temporary values and restore them before returning.

**Callee saved convention**: %rbx, %rbp, %r12 - %r15. When procedure P calls Q, Q must preserve the register value.

**Caller saved convention**: All other registers except %rsp. They can be modified by any function. Procedure P must save the data if it wants to use again.

**Callee save example**

```
long call_incr2(long x) {
	long v1 = 15213;
	long v2 = incr(&v1, 3000);
	return x+v2;
}

call_incr2:
	pushq %rbx            # Store whatever value before usage
	subq  $16, %rsp       # Allocate 16 bytes on the stack
	movq  %rdi, %rbx
	movq  $15213, 8(%rsp)
	movl  $3000, %esi
	leaq  8(%rsp), %rdi
	call  incr
	addq  %rbx, %rax
	addq  $16, %rsp       # Move back the stack pointer
	popq  %rbx            # Put back whatever value after usage
	ret
```

**Recursive call example**

```
long pcount_r(unsigned long x) {
	if (x == 0)
		return 0;
	else
		return (x & 1) + p_count_r(x >> 1);
}

pcount_r:
	movl   $0, %eax;
	testq  %rdi, %rdi;  # Check if equal zero
	je     .L6
	pushq  %rbx         # Going to use %rbx so save in stack first
	movq   %rdi, %rbx
	andl   $1, %ebx
	shrq   %rdi         # Shift right by 1
	call   pcount_r
	addq   %rbx, %rax
	popq   %rbx	
.L6
	rep; ret            # Just means return
```

**Array allocation**: Continguously allocated region of `L * sizeof(T)` bytes in memory. Array references are pointers.

Declaring `int val[5]` will allocate this region in memory.

```
 |   1   |   5   |   2   |   1   |   3
 x      x+4     x+8     x+12    x+16
```

Reference | Type | Value
------ | ------- | ------- 
val[4] | int | 3
val | int * | x
val + 1 | int * | x + 4
&val[2] | int * | x + 8

```
typedef int zip_dig[5];

void zincr(zip_dig z) {
	size_t i;
	for (i=0; i < 5; i++)
		z[i]++
}

zincr:
	movl  $0, %eax            # i = 0
	jmp   .L3
.L4
	addl  $1, (%rdi, %rax, 4) # z[i]++
	addq  $1, %rax            # i++
.L3
	cmpq  $4, %rax
	jbe   .L4
	rep; ret
```

**Nested Array Element**: Array A has R rows and C columns. If A[i][j] is an element that requires K bytes, its address is at `A + K(i*C + j)`.

**Multi-level Array**: Each item in the top array is a pointer of 8 bytes (64-bit).

```
int pgh[3][5];

int *univ[3] = {<arr1>, <arr2>, <arr3>}

Access pgh[index][digit]:
	Mem[pgh + 20*index + 4*digit]
	
Access univ[index][digit]:
	Mem[Mem[univ + 8*index] + 4*digit]
```

**Variable array**: Array can be declared either as a local variable or an argument to a function.

```c
int var_ele(long n, int A[n][n], long i, long j) {
	return A[i][j];
}
```

**Struct**: Represented as a block of memory big enough to hold all the fields.

```
struct rec {
	int a[3]; // Embedded within the struct
	int i;
	struct rec *next;
}

void set_val(struct rec *r, int val) {
	while(r) {
		int i = i->i;
		r->a[i] = val;
		r = r->next;
	}
}


.L11                              # loop
	movsq  16(%rdi), %rax         # i = M[r+16]
	movl   %esi, (%rdi, %rax, 4)  # M[r + 4*i] = val
	movq   24(%rdi), %rdi         # r = M[r+24]
	testq  %rdi, %rdi             # Test r
	jne    .L11
```

**Data alignment**: Primitive data types have K bytes. Addresses must be a multiple of K. For largest alignment requirement of K, the overall structure must be a multiple of K.

**Save space**: Rearrange the items in structures to save space. Place the biggest items in the beginning.

**FP basics**: All numbers are in %xmm registers. All registers are caller-saved. %xmm0 is for return.

**Union**: Allows a single object to be referenced according to multiple types. An enum can be added alongside to differentiate the types during runtime.

<ins>Union allocation</ins>: Only allocate memory for the largest element.

```
typedef union {
	float f;
	unsigned u;
} bit_float_t;

float bit2float(unsigned u) {
	bit_float_t arg;
	arg.u = u;
	return arg.f;
}
```

<ins>x86-64 Byte ordering</ins>

```
# Little Endian: Least significant byte has lowest address.

union {
	unsigned char c[8];
	unsigned int i[2];
} dw;

char 0-7 == [0xf0, 0xf1, 0xf2, 0xf3, 0xf4, 0xf5, 0xf6, 0xf7]
int  0-1 == [0xf3f2f1f0, 0xf7f6f5f4]
```

![](images/Pasted%20image%2020211211191215.png)

**x86-64 memory layout**

```
00007FFFFFFFFFFF -> (47-bit max)
Stack -> (Can go down max 8MB)
 |
 v

Shared Libraries

 ^
 |
Heap -> (Dynamically allocated as needed, malloc/ new)
Data -> (Statically allocated data, global/static/constants)
Text -> (Executable machine instructions)
```

**Vulnerable buffer code**

```
void echo() {
	char buf[4]; /* allocates 4 bytes */
	gets(buf);   /* reads a line from stdin and stores it */
	puts(buf);   /* writes to stdout up to (not including) the null char */
}

void call_echo() {
	echo();
}


# We are able to write 23 characters to 'echo'
# Writing 24 characters cause segmentation fault

echo:
	sub    $0x18, %rsp   # Allocates 24 bytes (20 extra)
	mov    $rsp, $rdi
	callq  <gets>
	mov    %rsp, %rdi
	callq  <puts>
	add    $0x18, %rsp
	retq
call_echo:
	sub    $0x8, %rsp
	mov    $0x0, %eax
	callq  <echo>
	add    $0x8, %rsp
	ret
```

Typing 23 characters (24 bytes) work fine because of unused space in stack. Typing 24 characters corrupts the return address.

![](images/Pasted%20image%2020211211174154.png)

**Worm**: Can run on its own and propagate.

**Virus**: Adds itself to other programs, cannot operate independently.

**Thawrting buffer overflow**

<ins>Randomized stack offsets</ins>: Allocate a random amount of space on stack. It is harder for the attacker to know where the code to execute is in the stack address.

<ins>Nonexecutable code segments</ins>: Virtual pages containing stack memory can be marked as non-executable.

<ins>Stack canary</ins>: Place a special value (canary) on stack beyond buffer and check for corruption before exiting function. Default.

```
# Canary

echo:
	sub    $0x18, %rsp
	mov    %fs:0x28, %rax  # Reading from read-only segment
	mov    %rax, 0x8(%rsp) # Place sth at 8-byte offset
	
	...
	
	mov    0x8(%rsp), %rax
	xor    %fs:0x28, %rax  # Check if corrupted
	je     <echo+0x39>
	callq  <__stack_chk_fail@plt> # Dump if corrupted
	...
	retq
```

**Return-oriented programming attacks**: Use existing library codes (Gadgets) to achieve a desired outcome. Gadgets typically end with a return instruction.