# SPHINCS+ in Cryptol

[SPHINCS+](https://sphincs.org) is an asymmetric hash-based signature scheme developed for the NIST post-quantum project.

This Markdown document is *literate Cryptol*,
and can be loaded like any other Cryptol module.
Cryptol only sees the contents of triple-backtick code blocks;
all other text here is strictly for human eyes.

```
module SphincsPlus where
```

The present document's heading structure follows the SPHINCS+ [specification](https://sphincs.org/data/sphincs+-specification.pdf),
and can be read as a supplement to that document.

---

## 2. Notation

This section shows the Cryptol equivalent of each basic operation used in the spec.

### 2.1 Data Types

Byte string literals are written as "0x" followed by a positive even number of hexadecimal digits,
as in `0xe534f0`.
Arrays of byte strings have uniform element length, determined by the longest element:
`[0x34, 5, 6] == [0x34, 0x05, 0x06]`.
Cryptol's built-in bit-string and array types are big-endian.

### 2.2 Functions

Cryptol doesn't have even an approximate notion of real or rational numbers,
and thus lacks *floor* and *ceiling* functions.
All the built-in operations on integer values return integers.

In particular, the base-2 logarithm returns the *ceiling* of its real-valued equivalent:
`(lg2 7, lg2 8, lg2 9) == (3, 3, 4)`.

| **Name** | **Spec syntax** | **Cryptol syntax** | **Example** |
| -------- | --------------- | ------------------ | ----------- |
| base-2 logarithm | ⌈ *log(x)* ⌉ | `lg2 x`     | `lg2 15 == 4` |

Truncation of a bit string is accomplished with Cryptol's `take` function,
writing *Trunc<sub>len</sub>(x)* as "take\`{*len*} *x*",
where "*len*" (in bits) is the function's first type-level parameter:
<pre>take`{16} 0x12345678 == 0x1234</pre>

### 2.3 Operators

The usual integer operators mostly have familiar notations,
and respect the standard precedence ordering.
Note that Cryptol has no rational numbers, so `/` means integer division.

| **Name** | **Spec syntax** | **Cryptol syntax** | **Example** |
| -------- | --------------- | ------------------ | ----------- |
| exponent | *a<sup>b</sup>* | `a ^^ b`           | `4 ^^ 3 == 64` |
| product  | *a · b* or *ab* | `a * b`            | `4 * 3 == 12` |
| integer quotient | ⌊ *a / b* ⌋ | `a / b`        | `5 / 3 == 1` |
| remainder | *a % b*        | `a % b`            | `5 % 3 == 2` |
| sum      | *a + b*         | *a + b*            | `4 + 3 == 7` |
| difference  | *a - b*      | *a - b*            | `3 - 4 == -1` |

The postfix `++` increment operator from C-like languages does not exist in Cryptol,
and can't be simulated by the divergent recursive binding `let a = a + 1`,
where variable `a` is already bound.
That Cryptol expression defines an infinite integer,
which cannot be evaluated in any useful way.
Instead of a destructive update, we must use a fresh variable: `let a' = a + 1`.

Cryptol's "logical" shift operators are defined on bit strings
rather than integers, a distinction not made by the spec:

| **Name** | **Spec syntax** | **Cryptol syntax** | **Example** |
| -------- | --------------- | ------------------ | ----------- |
| left shift  | *a << b* | `a << b` | `0x00ff00 << 8 == 0xff0000` |
| right shift | *a >> b* | `a >> b` | `0x00ff00 >> 8 == 0x0000ff` |

Array indexing is accomplished with Cryptol's infix `@` operator
rather than postfix square brackets.
Cryptol regards the byte strings defined above as
sequences of *bits* rather than bytes,
so we define another infix operator to index a whole byte.

```
(~@) : {n} (fin n) => [n * 8] -> [n] -> [8]
(~@) bytestring index = (split`{each=8} bytestring) @ index
```

| **Name** | **Spec syntax** | **Cryptol syntax** | **Example** |
| -------- | --------------- | ------------------ | ----------- |
| array index | *A[i]* | `A @ i` | `[0x0a, 0x0b, 0x0c] @ 1 == 0x0b` |
| byte string index | *X[i]* | `X ~@ i` | `0x0a0b0c ~@ 2 == 0x0c` |

Cryptol's bitwise logical operators for our byte strings
are written in a somewhat C-like syntax.

| **Name** | **Spec syntax** | **Cryptol syntax** | **Example** |
| -------- | --------------- | ------------------ | ----------- |
| bitwise conjunction | *A* AND *B* | `A && B` | `0xffff00 && 0x00ffff == 0x00ff00` |
| bitwise exclusive disjunction | *A* XOR *B* (or *A ⊕ B*) | `A ^ B` | `0xffff00 ^ 0x00ffff == 0xff00ff` |

Byte string concatenation is written with `#`.

| **Name** | **Spec syntax** | **Cryptol syntax** | **Example** |
| -------- | --------------- | ------------------ | ----------- |
| concatenation | *A \|\| B* | `A # B` | `0x0123 # 0xab == 0x0123ab` |

### 2.4 Integer to Byte Conversion

Cryptol uses a type-level function parameter
to determine the length of the resulting bit string.

| **Name** | **Spec syntax** | **Cryptol syntax** | **Example** |
| -------- | --------------- | ------------------ | ----------- |
| conversion | `toByte`(*x,y*) | `(fromInteger x) : [y*8]` | `((fromInteger 255) : [3*8]) == 0x0000ff` |

This parameter can also be written more explicitly:
<pre> fromInteger`{[y*8]} x </pre>

### 2.5 Strings of Base-*w* Numbers

The specified function does two things:
splitting a byte string into an array, and truncating the array.
Since our "byte strings" are really bit strings,
and *w* is always one of three specific powers of 2,
we can just use type-level arguments to Cryptol's built-in `split` and `take`.

| **Name** | **Spec syntax** | **Cryptol syntax** | **Example** where X = 0x1234 |
| -------: | --------------- | ------------------ | ----------- |
| base   4 | `base_w(X, 4, out_len)`   | take\`{out_len} (split\`{each=2} X) | out_len = 4 ==> [0x0, 0x1, 0x0, 0x2] |
| base  16 | `base_w(X, 16, out_len)`  | take\`{out_len} (split\`{each=4} X) | out_len = 4 ==> [0x1, 0x2, 0x3, 0x4] |
| base 256 | `base_w(X, 256, out_len)` | take\`{out_len} (split\`{each=8} X) | out_len = 1 ==> [0x1234] |

Note that the value of `split`'s argument `each` is the base-2 log of the spec's function parameter `w`.

### 2.6 Member Functions

Cryptol has record types, and uses a dot syntax for accessing record components.

| **Name** | **Spec syntax** | **Cryptol syntax** | **Example** where PK = {X = 0x12, Y = 0x34} |
| -------: | --------------- | ------------------ | ----------- |
| get member | `PK.getX()`     | `PK.X`             | `PK.X == 0x12` |
| set member | `PK.setX(new_val)` | `PK' = {X = new_val, ... } ` | `PK' = {X = 0x56, Y = PK.Y}` |

Instead of destructive updates, we must construct a whole new record and give it a new name.

### 2.7 Hash Function Families

See Section 7 for the definition of specific *tweakable* hash functions,
which abstract over the implementations of SHA2 or SHAKE.

#### 2.7.1 Tweakable Hash Functions

Translating the given function signatures:

<pre>
T_l : {l, n} (fin n, fin l, Arith n, Arith l) =>
      [n] -> [32] -> [l*n] -> [n]
//   PK.seed ADRS      M      md
F = T_l`{1}
H = T_l`{2}
</pre>

#### 2.7.2 PRF and Message Digest

Specific pseudo-random functions will be defined (or imported?) later.
Translating their signatures, and guessing at the type constraints:

<pre>
PRF : {n} (fin n) => [n] -> [32] -> [n]

PRF_msf : {n, x} (fin n) => [n] -> [n] -> [x] -> [n]

H_msg : {n, x, m} (fin n, fin m) => [n] -> [n] -> [n] -> [x] -> [m]
</pre>

#### 2.7.3 Hash Function Address Scheme

To facilitate working with the five different types of address,
we define a single record form having all the fields of each,
along with functions for converting between these records
and flat byte string representations.

```
type ADRS        = [256]
type AddressWord = [32]
type TreeAddress = [96]

WOTS_HASH  = 0 : AddressWord
WOTS_PK    = 1 : AddressWord
TREE       = 2 : AddressWord
FORS_TREE  = 3 : AddressWord
FORS_ROOTS = 4 : AddressWord

type Address =                // 0 1 2 3 4
    { layer    : AddressWord  // X X X X X 
    , tree     : TreeAddress  // X X X X X
    , _type    : AddressWord  // X X X X X
    , keypair  : AddressWord  // X X
    , chain    : AddressWord  // X
    , hash     : AddressWord  // X
    , height   : AddressWord  //     X X
    , index    : AddressWord  //     X X
    }

addressBytes : Address -> ADRS
addressBytes addr = addr.layer # addr.tree # t # rest
    where
    t = addr._type
    rest =
        if t == WOTS_HASH  then  addr.keypair # addr.chain # addr.hash
         | t == WOTS_PK    then  addr.keypair # zero
         | t == TREE       then  zero # addr.height # addr.index
         | t == FORS_TREE  then  addr.keypair # addr.height # addr.index
         | t == FORS_ROOTS then  addr.keypair # zero
                           else  error "bad address type"

addressRecord : ADRS -> Address
addressRecord adrs =
    { layer   = layer'
    , tree    = word1 # word2 # word3
    , _type   = _type'
    , keypair = if is_wh \/ is_pk then word5 else zero
    , chain   = if is_wh          then word6 else zero
    , hash    = if is_wh          then word7 else zero
    , height  = if is_tree        then word6 else zero 
    , index   = if is_tree        then word7 else zero
    }
    where
    [layer', word1, word2, word3,
     _type', word5, word6, word7] = split`{each=32} adrs
    is_wh     = _type' == WOTS_HASH
    is_pk     = _type' == WOTS_PK
    is_tree   = _type' == TREE \/ _type' == FORS_TREE

setAddressType : AddressWord -> Address -> Address
setAddressType new_type adrs =
    { layer    = adrs.layer
    , tree     = adrs.tree
    , _type    = new_type
    , keypair  = zero
    , chain    = zero
    , hash     = zero
    , height   = zero
    , index    = zero
    }

```

**TODO**