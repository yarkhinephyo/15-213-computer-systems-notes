**Hexadecimal**: 1010 -> A, 1100 -> C, 1111 -> F

**Word size**: Nominal size of pointer data. _64-bit_ machine means 8-byte addresses and _32-bit_ machine means 4-byte addresses.

**Fixed-size integer types**: Using `int32_t`, `uint32_t`, `int64_t` instead of `int`, `unsigned int`, `long` respectively ensures that data sizes are independent of compilers or machines.

**Shift operations**

<ins>Left Shift</ins>: Fill 0 on the right.

<ins>Right Shift (Logical)</ins>: Fill 0 on the left.

<ins>Right Shift (Arithmetic)</ins>: Fill the most significant bit on the left.

Almost all compilers use the arithmetic version for signed data. For unsigned data, however, rights shifts must be logical.

Bit shifting operates on numbers in the register. This is <ins>independent</ins> of the byte ordering in the memory.

**Unsigned values**: Bit positions multiply with the corresponding powers of 2.

**Two's complement values**: The most left bit is a negative value.

Type | Decimal | Hex
------ | ------- | ------ 
UMax | 65535 | FF FF
TMax | 32767 | 7F FF
TMin | -32768 | 80 00
-1 | -1 | FF FF
0 | 0 | 00 00

**Mappings between unsigned and two's complement**: Keep the bit representations and reinterpret. Interpretation may change but the bit representation remains the same.

For nonnegative values, the encodings are the same. Negative values can be easily converted to/from unsigned format with a difference of 2<sup>w</sup>. This can be thought of as 2<sup>w-1</sup> to negate the first bit and another 2<sup>w-1</sup> to add the positional value.

```
Signed    FF FF -> -1
Unsigned  FF FF -> -1 + (2 ^ 16) -> 65535
```

![](images/Pasted%20image%2020220429152114.png)

**Casting surprises**: If there is a mix of two's complement and unsigned, values are implicitly cast to unsigned.

```
-1 < 0
-1 > 0U
-1 > -2 
(unsigned) -1 > -2
```

Infinite loop bug in the example below.

```c
// First example
unsigned i;
for (i=n-1; i>=0; i--) {
	func(a[i])
}

// Second example
int i;
for (i=n-1; i - sizeof(int) >= 0; i--) {
	func(a[i])
}
```

**Extension**:

<ins>Unsigned</ins>: Zero extension.

<ins>Two's complement</ins>: Sign extension. Copy the most significant bit.

```
  1 1 1 0 -> -8 + 4 + 2 = -2
1 1 1 1 0 -> -16 + 8 + 4 + 2 = -2
```

**Truncation**

<ins>Unsigned</ins>: x' = x mod 2<sup>k</sup> where truncation target is k-bits.

<ins>Two's complement</ins>: Convert to unsigned for truncation. Then convert back to two's complement. x' = U2T( T2U(x) mod 2<sup>k</sup> ). 

**Addition**

<ins>Unsigned</ins>: Add and drop the overflow. UAdd(u, v) = (u + v) mod 2<sup>w</sup>. Overflow has occurred if the result is less than either of the numbers. 

<ins>Two's complement</ins>: TAdd has the same bit level behaviour as UAdd. TAdd = U2T( UAdd( T2U(x), T2U(y) ) ). Positive overflow has occurred if x > 0 and y > 0 but s <= 0. Negative overflow has occurred if x < 0 and y < 0 but s >= 0.

```
// Negative overflow
// -9 cannot be represented with Signed 4-bit
1 1 0 1 -> -3
1 0 1 0 -> -6
-------
0 1 1 1 -> +7

// Positve overflow
0 1 1 1 -> +7
0 1 0 1 -> +5
-------
1 1 0 0 -> -4
```

**Negation**

<ins>Unsigned</ins>: 2<sup>w</sup> - x if x > 0. You can get 0 if you do unsigned addition with the original number.

<ins>Two's complement</ins>: -x if x != TMin else TMin. In a two's complement system,  `-x == ~x + 1`. There is always the corner case of TMin that should be checked during testing.

**Multiplication**

<ins>Unsigned</ins>: Multiply and drop the overflow. UMult(u, v) = (u \* v) mod 2<sup>w</sup>.

<ins>Two's complement</ins>: Similar to addition, bit level equivalence of two's complement and unsigned multiplications are identical. TMult(u, v) = U2T( (u \* v) mod 2<sup>w</sup> ).

**Multiply Power-of-2**: Bit-shift left.

**Divide Power-of-2**: Bit-shift right (Arithmetic) rounds towards zero.

**Counting Down with Unsigned**

```c
unsigned i;
for (i = cnt-2; i < cnt; i--) {
	a[i] += a[i+1]
}
```

**Byte ordering**: Addresses specify first byte locations. Addresses of successive words differ by 4-bytes (32-bits) and 8-bytes (64-bits).

<ins>Little Endian</ins>: Least significant byte has lowest address. Common in most Intel-compatible machines.

<ins>Big Endian</ins>: Least significant byte has highest address. Common in machines from IBM and Oracle.

```
// Variable x has 4-byte of 0x01234567

// Big Endian
0x100|0x101|0x102|0x103|
01   |23   |45   |67   |

// Little Endian
0x100|0x101|0x102|0x103|
67   |45   |23   |01   |
```

**Representing Strings**: Array of characters with a final character of 0. All machines have the same order.

```
char S[6] = "18213";

0x31 | 0x38 | 0x32 | 0x31 | 0x33 | 0x00
```

**C Puzzles**

```
int x,y;
unsigned ux, uy;
ux = x;
uy = y;

x < 0 -> ((x*2) < 0)        [False statement, consider TMin]
ux >= 0                     [True Statement]
x & 7 == 7 -> (x << 30) < 0 [True Statement]
ux > -1                     [False Statement]
x > y -> -x < -y            [False Statement, consider TMin]
x * x >= 0                  [False Statement, consider overflow]
x >= 0 -> -x <= 0           [True statement, all +ve have inverse]
x <= 0 -> -x >= 0           [False Statement, consider TMin]
(x|-x) >> 31 == -1          [False Statement, only False for 0]
                             # Note: Signed int has arithmetic shift
```