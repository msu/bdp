# The Byte Data Processor 1 (BDP-1)

The BDP-1 is a simple 8-bit learning computer inspired by the [Little Man Computer](https://en.wikipedia.org/wiki/Little_Man_Computer).  

It is designed to complement the [MTMC-16](https://mtmc.cs.montana.edu) as a simpler introduction to computing.

# Architecture

- 8 bit words
- 4 general registers: A, B, C, D
- 256 bytes of memory
- 16 x 16 display, 8-bit color ([RGB332](https://roger-random.github.io/RGB332_color_wheel_three.js/))

## Register conventions

- `A` - accumulator: destination of RMATH/RLOGIC; tested by JZ/JP/JN; result register for WINT/RINT/RAND
- `B`, `C` - general purpose, but PLOT uses as x and y coordinates (masked to 4 bits)
- `D` - general purpose, but PLOT uses as color

# Instruction Set

* Top nibble is the instruction type
* Bottom nibble is op code or args
* The MSB of the top nibble distinguishes single-word (0xxx) from double-word (1xxx) instructions
* The assembler uses unified `MOV dest, src` syntax (Intel order): 
  * Brackets mean memory access
  * Bare names/numbers are immediate

| Type   | Top  | Op   | Assembly form  | Size   | Description                                                       |
|--------|------|------|----------------|--------|-------------------------------------------------------------------|
| SYS    | 0000 | 0000 | HALT           | single | halts computer                                                    |
| SYS    | 0000 | 0001 | WINT           | single | writes A to standard out as an unsigned integer                   |
| SYS    | 0000 | 0010 | RINT           | single | reads standard in to A as an unsigned integer                     |
| SYS    | 0000 | 10RR | RAND RR        | single | places a random number in register RR                             |
| SYS    | 0000 | 1111 | NOOP           | single | no operation                                                      |
| GFX    | 0001 | 0000 | CLEAR          | single | resets graphics to blank (all 0)                                  |
| GFX    | 0001 | 0001 | PLOT           | single | sets pixel at position B, C to color in D                         |
| MOV    | 0010 | SSDD | MOV DD, SS     | single | DD := SS (register-to-register copy)                              |
| MOV    | 0011 | PPDD | MOV DD, [PP]   | single | DD := M[PP]  (indirect load via pointer in PP)                    |
| MOV    | 0100 | PPSS | MOV [PP], SS   | single | M[PP] := SS (indirect store via pointer in PP)                    |
| RMATH  | 0101 | 00RR | ADD A, RR      | single | A += RR (accumulator add)                                         |
| RMATH  | 0101 | 01RR | SUB A, RR      | single | A -= RR                                                           |
| RMATH  | 0101 | 10RR | MUL A, RR      | single | A *= RR                                                           |
| RMATH  | 0101 | 11RR | DIV A, RR      | single | A /= RR (integer; div by 0 -> 0)                                  |
| RUNARY | 0110 | 00RR | INC RR         | single | RR += 1                                                           |
| RUNARY | 0110 | 01RR | DEC RR         | single | RR -= 1                                                           |
| RUNARY | 0110 | 10RR | NEG RR         | single | RR := -RR (two's complement)                                      |
| RUNARY | 0110 | 11RR | ZERO RR        | single | RR := 0                                                           |
| RLOGIC | 0111 | 00RR | AND A, RR      | single | A &= RR                                                           |
| RLOGIC | 0111 | 01RR | OR  A, RR      | single | A \|= RR                                                          |
| RLOGIC | 0111 | 10RR | XOR A, RR      | single | A ^= RR                                                           |
| RLOGIC | 0111 | 11RR | SHL A, RR      | single | A <<= RR (shift left, fills 0)                                    |
| MEM    | 1000 | 00RR | MOV RR, [addr] | double | RR := M[addr]                                                     |
| MEM    | 1000 | 01RR | MOV RR, imm    | double | RR := imm (the second byte is the literal value)                  |
| MEM    | 1000 | 10RR | MOV [addr], RR | double | M[addr] := RR                                                     |
| MMATH  | 1001 | 00RR | ADD RR, [addr] | double | RR += M[addr]                                                     |
| MMATH  | 1001 | 01RR | SUB RR, [addr] | double | RR -= M[addr]                                                     |
| MMATH  | 1001 | 10RR | MUL RR, [addr] | double | RR *= M[addr]                                                     |
| MMATH  | 1001 | 11RR | DIV RR, [addr] | double | RR /= M[addr] (integer; div by 0 -> 0)                            |
| JUMP   | 1010 | 0000 | JMP addr       | double | PC := addr                                                        |
| JUMP   | 1010 | 0001 | JZ  addr       | double | jump if A == 0                                                    |
| JUMP   | 1010 | 0010 | JP  addr       | double | jump if A > 0 (high bit clear, value non-zero)                    |
| JUMP   | 1010 | 0011 | JN  addr       | double | jump if A < 0 (high bit set)                                      |
| JUMP   | 1010 | 0100 | JSR addr       | double | writes return address into M[addr], then jumps to addr+1          |
| JUMP   | 1010 | 0101 | JI  addr       | double | PC := M[addr] (indirect jump; serves as RET when paired with JSR) |
| LOGIC  | 1011 | 00RR | AND RR, [addr] | double | RR &= M[addr]                                                     |
| LOGIC  | 1011 | 01RR | OR  RR, [addr] | double | RR \|= M[addr]                                                    |
| LOGIC  | 1011 | 10RR | XOR RR, [addr] | double | RR ^= M[addr]                                                     |
| LOGIC  | 1011 | 11RR | SHL RR, [addr] | double | RR <<= M[addr] (shift left, fills 0)                              |

## Subroutine convention

The BDP uses a [PDP](https://en.wikipedia.org/wiki/PDP-8) style for invoking sub-routines: 

* `JSR slot` writes the return address into `M[slot]` and jumps to `slot+1`
* The function body lives at `slot+1`
* Return with `JI slot` (indirect jump through the saved address) 
* NO STACK: not re-entrant!

```
        MOV A, 5
        JSR double      ; M[double] := return addr; PC := double+1
        WINT
        HALT
double: SW 0             ; return slot; body starts at the next byte
        MOV [scratch], A
        ADD A, [scratch]
        JI  double       ; return
scratch: SW 0
```

## Assembler directives

* `label:` defines a label at the current address (no `.org` - the assembler emits code sequentially starting at 0x00)
* `SW value [, value ...]` emits one raw byte per value
* `;` starts a comment to end of line

# ByteTran

ByteTran is a Fortran-flavored high level language that compiles to BDP-1 assembly. It is uppercase, integer-only & whitespace-sensitive.

## Grammar

```
program     = { statement }
statement   = IDENT '=' expr             ! assignment
            | 'WRITE' expr
            | 'DO' 'WHILE' expr { statement } 'END'
            | 'IF' expr { statement } [ 'ELSE' { statement } ] 'END'
expr        = 'READ' | additive [ ('>='|'==') additive ]
additive    = primary [ ('+'|'-') primary ]
primary     = IDENT | NUMBER | 'TRUE' | 'FALSE'
comment     = '!' to end of line
```

## Example

```
! Fibonacci - read N, print fib(N)
N = READ
A = 0
B = 1
DO WHILE N >= 1
  T = A + B
  A = B
  B = T
  N = N - 1
END
WRITE A
```
