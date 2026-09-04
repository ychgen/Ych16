# Instruction Format

Ych16 instructions are variable-length whose size fall out naturally as the decoder eventually exhausts an instruction interpretation normally. A Ych16 instruction roughly looks like:  
`[Prefixes (optional)] [Opcode(s)] [Payloads (optional, depends on opcode and potentially cascades)]`  
Instructions have a minimum length of 1 (opcode only) and no upper limit.  
Prefixes contain any prefix within `00-0F` range.  
Opcode(s) means either a primary opcode or escape to secondary (55) or tertiary (AA) opcode spaces where the next byte is the opcode.  

Payloads may refer to:
- Opcode prerequisite payload (like IMM8/16, AMA, OTO DB, CRA DB, ShD etc.)
- Opcode prerequisite payloads that cause more payloads like OTO DB with fetchmusts or ShD and CRA DB with potential displacements.  

## General Rules

- In case source & destination semantics exist, Payloads itself is broken down as `[Source Payloads][Destination Payloads]`. All source payloads come first if any, all destination payloads follow, if any.
- All 8 bit immediates are zero-extended to 16-bits. This excludes CRA displacements as they are counted as displacements rather than an immediates. CRA displacement 8-bits is sign extended.

## Prerequisites & Postrequisites

Prerequisites are protooperand structures declared by an opcode. Postrequisites are operand resolvers, prerequisites define whether or not they need any postrequisites.

| Requisite | Operand Class | Fetch Required |
| :-------: | :-----------: | :------------: |
|    N/A    |      N/A      |       N/A      |
|   Imm8    |   Immediate   |      Byte      |
|   AMA     |   Memory      |      Word      |
|   Imm16   |   Immediate   |      Word      |
|   OTO     |   ?????????   | OTO DB (byte)  |
|   CRA     |   Memory      | CRA DB (byte)  |
|   ShD     |   Memory      |    ShD byte    |

### N/A

An opcode that has N/A prerequisite can only have one global N/A semantic like OTO. This means the opcode is operandless and only the opcode and prefixes are meaningful for the instructions, no operands will be decoded.  

Example instruction: `RTN n/a - Return`

### Imm8

Zero-extended 8 bit immediate.

Example instruction: `RTD imm8 - Return with Disclosure`

### AMA

Absolute memory addressing. Fetched word is the effective address (EA).

Example instruction: `CAL ama - Call`

### Imm16

Native 16 bit immediate.

### OTO (Operand-to-Operand)

OTO operand prerequisite is special. It must be declared at opcode level and no cascade can reproduce OTO as a postrequisite. An opcode can only declare that the whole opcode only has one prerequisite and such is OTO. No source/destination prerequisite separation, as OTO DB itself supplies both source and destination grammar that potentially trigger other postrequisites.  
OTO can result in the source and destination collapsing into any of the three operand classes; Register, Memory, Immediate.

Example instruction: `ADD oto - Add`

#### OTO DB (Operand-to-Operand Descriptor Byte)

| Bit Offset | Field Name  |
| :--------: | :---------: |
|     7-4    |   SRC UOD   |
|     3-0    |   DST UOD   |

NOTE: Even though the names are SRC and DST, those reflect traditional and common roles. Some instructions just need two operands, they might use both SRC and DST as regular operands rather than source and destination semantics. The ISA documentation will call them SRC and DST regardless.

##### UOD (Unit Operand Descriptor)

|        Pattern        |    Pattern Name    | Operand Class | Inner Bits Encode | Postrequisite |
| :-------------------: | :----------------: | :-----------: | :---------------: | :-----------: |
| xxx1 (excluding 1111) |  Register Direct   |   Register    |    Register ID    |      N/A      |
|         1111          | Shorthand Displace |    Memory     |    Ineligible.    |    ShD byte   |
|         1xx0          | Register Indirect  |    Memory     |  Base Register ID |      N/A      |
|         0xx0          |     Fetchmusts     |      TBD      |  See table below  |      TBD      |

| Fetchmust Inner Pattern |    Pattern Name    | Operand Class | Postrequisite |
| :---------------------: | :----------------: | :-----------: | :-----------: |
|           00            |        CRA         |   Memory      |    CRA DB     |
|           01            |        Imm8        |   Immediate   |     Byte      |
|           10            |        AMA         |   Memory      |     Word      |
|           11            |        Imm16       |   Immediate   |     Word      |

#### CRA DB (Complex Register Addressing Descriptor Byte)

Default mode (indexed):

| Bit Offset | Field |        Full Name         | Description | Encode |
| :--------: | :---: | :----------------------: | :---------: | ------ |
|     7-6    |  D/I  | Displacement/Indexedness | Encodes displacement validity, if so, size. Also able to alter into nonindexed form. | • `00` : No Disp<br> • `01` : Nonindexed Mode<br> • `10` : Displacement 8-bits<br> • `11` : Displacement 16-bits |
|     5-4    |  PTR  | Base Register            | The base register. | Base Register ID. |
|     3-2    |  IDX  | Index Register           | The scaled index register. | Index Register ID. |
|     1-0    |  MUL  | Scale Factor             | Index scalar factor. | Sequentially from lowest to highest, `{✕1,✕2,✕4,✕8}`. |

Nonindexed mode (D/I = `01`):

| Bit Offset | Field |        Full Name          | Description | Encode |
| :--------: | :---: | :-----------------------: | :---------: | ------ |
|     7-6    |  ---  | Mode Alter                | Value that caused nonindexed mode. | `01` |
|     5-4    |  PTR  | Base Register             | The scaled base register. | Base Register ID. |
|     3-2    |  D/R  | Displacement/Relativeness | Encodes displacement validity, if so, size. Also able to alter into RIP-relative form. | • `00` : No Disp<br> • `01` : RIP-relative Mode<br> • `10` : Displacement 8-bits<br> • `11` : Displacement 16-bits |
|     1-0    |  MUL  | Scale Factor              | Base scalar factor. | Sequentially from lowest to highest, `{✕1,✕2,✕4,✕8}`. |

RIP-relative mode (D/I = `01` && D/R = `01`):

| Bit Offset | Field |        Full Name          | Description | Encode |
| :--------: | :---: | :-----------------------: | :---------: | ------ |
|     7-6    |  ---  | Mode Alter                | Value that caused nonindexed mode. | `01` |
|     5-4    |  ---  | Reserved                  | Reserved, must be canonically zero. | `00` |
|     3-2    |  ---  | Mode Alter                | Value that caused RIP-relative mode. | `01` |
|     1-0    |  ---  | Reserved                  | Reserved, must be canonically zero. | `00` |

This always produces byte `44h` due to its nature. It also mandates a forced 16-bit displacement.

##### EA for Each CRA Mode

|     Mode     | Effective Addres (`EA`) Formula |
| :----------: | :-----------------------------: |
|    Indexed   |  Base + (Index ✕ Scale) + Disp  |
|  Nonindexed  |       Base ✕ Scale + Disp       |
| RIP-relative |           RIP + Disp            |

Assume `0` for Disp element of the formula if no displacement exists.

#### ShD (Shorthand Displacement)

ShD is a compact addressing primitive that is useful for accessing reasonably-sized stack locals and struct objects. It can displace up to `+64` bytes from a base register. It is denser than `CRA` + 8-bit displacement because that clocks at 2 bytes of encode while ShD is only one byte of encode.

| Bit Offset | Field Name | Description | Encodes |
| :--------: | :--------: | :---------: | :-----: |
|     7-6    |     PTR    | Base register. | Base register ID. |
|     5-0    |     DIS    | 6-bit **unsigned** displacement. | Actual displacement, minus one. This is because `000000` is treated as `+1` displacement, so counting starts from `1`. |

Any negative displacement or positive displacement exceeding `+64` must go through `CRA`'s nonindexed form with preferred displacement size encoded in `D/R`.
