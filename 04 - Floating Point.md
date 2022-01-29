**Fractional Binary Numbers**

Limited by numbers of the form x / (2 ^ k) for exact representation. Other rational numbers require repeating bit representations.

Limited by one setting of the binary point so there is a limited range of numbers. Floating point solves this issue.

```
bi  | bi-1  | ... | b2  | b1  | b0  | b-1  | ... | b-j
2^i |2^(i-1)| ... | 2^2 | 2^1 | 2^0 |2^(-1)| ... | 2^(-j)

// Example
5 3/4  -> 101.11
2 7/8  ->  10.111
1 7/16 ->   1.0111
```

**Floating Point Representation**

**Numerical form**: `(-1)s (M) * (2^E)` where "s" determines whether the number is -ve or +ve, "M" is a fractional value in [1.0, 2.0) and "E" weights value by a power of two.

<ins>32-bit</ins>: 1-bit (s), 8-bits (exp), 23-bits (frac)

<ins>64-bit</ins>: 1-bit (s), 11-bits (exp), 52-bits (frac)

**Normalized Values**:  Exp is not all zeros or ones. 

<ins>Significand (M)</ins>: Encoded with implied leading 1.

<ins>Exponent (E)</ins>: Requires calculation from exponent bits and the bias term.

```
E = exp - bias 
bias = 2^(k - 1) - 1 # where k is the number of "esp" bits.

Single precision: 127 (Exp: 1...254, E: -126...127)
Double precision: 1023 (Exp: 1...2046, E: -1022...1023)

// Example
float F = 15213.0
        = 11101101101101
		= 1.1101101101101 x 2 ^ (13)
		
frac = 11011011011010000000000 (23-bit frac)

bias = 127
exp = 140 = 10001100
E = exp - bias = 13
```

**Denormalized Values**: Exp is all zeros.

<ins>Significand</ins>: Encoded with implied leading 0.

<ins>Exponent</ins>: `E = 1 - Bias`.

**Infinity**: Exp is all ones and frac is all zeros. Floating points overflow to a special value called infinity.

**NaN**: Exp is all ones and frac is not all zeros.

**Number Line**: Distribution gets dense when closer to 0. 

```
|-inf|   - Normalized   | -Denorm |-0| 0| +Denorm |  +Normalized   |+inf|
```

**Comparison**: With the exception of special values, can be compared as Unsigned Integer.

**8-Bit Floating Point Example**

```
Bias = 7 (4-bit exp)

n: M = 1.xxx...
d: M = 0.xxx...
n: E = exp - bias
d: E = 1 - bias
```

![](images/Pasted%20image%2020211126232934.png)

**Round Nearest Even**: Round up or round down to closer integers. If exactly in the middle, round to the nearest even number. Example: 1.4 -> 1, 1.5 -> 2, -1.5 -> -2. Other rounding modes are statistically biased.

**Round Binary Numbers**
For binary numbers, "even" occurs when the least significant bit is zero. "Halfway" occurs when bits to the right are "100....".

```
// Round to nearest 1/4 (2-bits right)

Value   Binary    Rounded  Rounded Value
2 3/2   10.00011  10.00    2
2 3/16  10.00110  10.01    2 1/4
2 7/8   10.11100  11.00    3
2 5/8   10.10100  10.10    2 1/2
```

**Properties of FP Addition**: Commutative but not associative.

**Properties of FP Multiply**: Commutative but not associative. Multiplication does not distributes over addition.

**Floating Point in C**

Unlike between "int" and "unsigned", casts between "float", "int", "double" changes the bit representations.

1. Float cast to an Int may not need any rounding. (23-bit to 32-bit)
2. Double cast to an Int requires some rounding. (52-bit to 32-bit)
3. Int to double does not lose any bits. (32-bit to 52-bit)
4. Int to float requires some rounding. (32-bit to 23-bit)

**Floating Point Puzzles**

```
int x;
float f;
double d;

x == (int)(float) x      -> False
x == (int)(double) x     -> True
f == (float)(double) f   -> True
d == (double)(float) d   -> False
f == - (-f)              -> True   (Only toggling bit)
2/3 == 2/3.0             -> False  (Int & float cast to float)
d < 0.0 => ((d*2) < 0.0) -> True   (Even overflow will be -INF)
d > f   => -f > -d       -> True
d * d >= 0.0             -> True
(d+f) - d == f           -> False
```
