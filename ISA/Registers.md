# Registers

The Ych16 ISA features 7 GPRs (general-purpose registers), each having the same size as the word size of the ISA, 16-bits. A detailed table below showcases every register and their ordinary properties:

| Name |        Full Name       | Indirectible | Indexible |
| :--: | :--------------------: | :----------: | :-------: |
|  RA  | Register Accumulator   |      ✓       |     ✓     |
|  RB  | Register Base          |      ✓       |     ✗     |
|  RC  | Register Counter       |      ✗       |     ✓     |
|  RD  | Register Data          |      ✗       |     ✗     |
|  RG  | Register General       |      ✓       |     ✓     |
|  RI  | Register Index         |      ✗       |     ✓     |
|  RS  | Register Stack Pointer |      ✓       |     ✗     |

Do note that all GPRs are directible & mutable, so ordinary register direct can target any of them.

## Base Register IDs

If any ordinary GPR is named using 3-bit encode, the table below has each register's corresponding encoding:

| Register | Encoding |
| :------: | :------: |
|    RA    |   000    |
|    RB    |   001    |
|    RC    |   010    |
|    RD    |   011    |
|    RG    |   100    |
|    RI    |   101    |
|    RS    |   110    |

Encoding `111` is usually something different or unused. If it must correspond to a register, `111` always causes `0` semantically.  
For example, encoding `111` for register ID within a `UOD (Unit Operand Descriptor)` causes `ShD (Shorthand Displace)`, not any register.

## Indirect Register IDs

Base registers are named using 2-bit encode, the table below has each register's corresponding encoding: 

| Register | Encoding |
| :------: | :------: |
|    RA    |    00    |
|    RB    |    01    |
|    RG    |    10    |
|    RS    |    11    |

## Index Register IDs

Index registers are named using 2-bit encode, the table below has each register's corresponding encoding: 

| Register | Encoding |
| :------: | :------: |
|    RA    |    00    |
|    RC    |    01    |
|    RG    |    10    |
|    RI    |    11    |
