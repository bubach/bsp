# BSP specification

A binary scripted patch, hereafter a BSP, is a file containing a script indicating how to convert some data, called the
source, into some other data, called the target. In order to achieve this goal, the patcher must execute the script
contained in the BSP. This document formally specifies how that execution must be done, and how the BSP file must be
interpreted.

The words "byte", "halfword" and "word" throughout this document respectively refer to 8, 16 and 32-bit quantities. All
halfword or word quantities are stored, read and written in little-endian form (that is, least significant byte first)
unless otherwise indicated.

For clarity of exposition, in some cases, the patch source syntax (as interpreted by this project's compiler) is used.
This is done purely for simplicity; the patch source syntax does not constitute part of this specification, and any
other implementation of a BSP compiler may use any other syntax or even a completely different language compiled to the
BSP format. The BSP format is fully specified by its binary form as documented here.

## Execution model

A BSP engine must be capable of executing a BSP file according to this model. The model describes how the engine
behaves and what a BSP can expect of the engine running it.

The execution model consists of the following elements:

* A patch space, which contains the contents of the BSP file; this space is initialized when loading the BSP file. The
  patch space is immutable (that is, read-only and of fixed size), and it has the same size as the BSP file itself.
* A file buffer, which is the memory space where the operations of the BSP file are carried out. This file buffer is of
  variable size, and it is initialized with the data of the source file; it has the same size as the source file upon
  initialization.
* 256 word variables, all of which are initialized to zero. These variables are numbered 0 to 255 and can be referenced
  by the instructions in the BSP.
* An instruction pointer, which is a word initialized to zero.
* A current file pointer, which is a word initialized to zero and in unlocked state. The current file pointer can be in
  one of two states: unlocked, in which case it behaves normally as documented here, and locked, which makes it a
  read-only value; writes or updates to the current file pointer must be silently discarded (without causing any
  warnings or other kinds of error messages) while in locked state.
* A stack, unbounded in size, that contains word values. It is initialized empty.

The patch space and the file buffer have a maximum size of 4,294,967,295 bytes. Attempting to write to the file buffer
past its end increases its length (zero-filling any gaps that occur this way); attempting to read from either the file
buffer or the patch space beyond their respective ends is an error. Note that executing code involves reading from the
patch space, and thus this restriction still applies.

A location in the patch space is referred to as an _address_, while a location in the file buffer is referred to as a
_position_ (therefore, the value of the instruction pointer is an address, and the value of the current file pointer is
a position). Both of them are numbered sequentially from zero in the usual way.

All arithmetic is carried out in 32-bit words unless otherwise noted. All values are treated as unsigned other than
when indicated; negative values are simply a notational convenience (e.g., -1 is actually `0xffffffff`, that is, the
same as 4,294,967,295).

All errors should be treated as fatal. If the engine encounters an error condition (such as an invalid opcode, reading
past the end, or a division by zero), it should not attempt to resume execution; instead, it should inform the user of
the error and abort immediately.

The execution of the patch results in a word value called an exit status, similar in spirit to the exit status of a
program in several operating systems. An exit status of zero indicates success; other values must be treated as failure
and thus handled as errors. When a BPS exits with a status of zero, the contents of the file buffer must be written out
to the target file as the output of the patching process.

Execution of the BPS proceeds one instruction at a time, in the usual way for this format: the byte pointed by the
instruction pointer is read, which determines the opcode; the operands (if any) for that opcode are read as well; and
the instruction pointer is incremented by the total amount of bytes read; the instruction is then executed. Execution
continues this way until the BPS exits (via the corresponding instruction). Note that some instructions will modify
the instruction pointer upon execution.

## Opcodes

The opcode of each instruction is defined by the first byte. This also defines its length, and the kind of operands it
will take. An instruction can be asigned more than one opcode depending on its arguments: for instance, the `set`
instruction has two opcodes; one when called with two variables as its operands, and another one when called with a
variable and an immediate (a constant value encoded in the instruction itself).

Variables are always encoded as a single byte indicating the variable number. Immediates (that is, constant numerical
operands that are encoded directly in the instruction) are typically words, and they are encoded in little-endian
format; the actual width of immediate operands is indicated in the opcode table. Operands are encoded in the order that
they appear.

Note that the size of an instruction (in bytes) can be calculated by adding the size of its operands and adding one
byte for the opcode itself. It is shown in the table for simplicity.

Opcode bytes that don't appear in the following table constitute undefined instructions, and attempting to execute one
of them must be considered a fatal error.

|Opcode byte|Instruction           |Size|Operands                                |
|:---------:|:---------------------|---:|:---------------------------------------|
|`0x00`     |nop                   |   1|none                                    |
|`0x01`     |return                |   1|none                                    |
|`0x02`     |jump                  |   5|word                                    |
|`0x03`     |jump                  |   2|variable                                |
|`0x04`     |call                  |   5|word                                    |
|`0x05`     |call                  |   2|variable                                |
|`0x06`     |exit                  |   5|word                                    |
|`0x07`     |exit                  |   2|variable                                |
|`0x08`     |push                  |   5|word                                    |
|`0x09`     |push                  |   2|variable                                |
|`0x0a`     |pop                   |   2|variable                                |
|`0x0b`     |length                |   2|variable                                |
|`0x0c`     |readbyte              |   2|variable                                |
|`0x0d`     |readhalfword          |   2|variable                                |
|`0x0e`     |readword              |   2|variable                                |
|`0x0f`     |pos                   |   2|variable                                |
|`0x10`     |getbyte               |   6|variable, word                          |
|`0x11`     |getbyte               |   3|variable, variable                      |
|`0x12`     |gethalfword           |   6|variable, word                          |
|`0x13`     |gethalfword           |   3|variable, variable                      |
|`0x14`     |getword               |   6|variable, word                          |
|`0x15`     |getword               |   3|variable, variable                      |
|`0x16`     |checksha1             |   6|variable, word                          |
|`0x17`     |checksha1             |   3|variable, variable                      |
|`0x18`     |writebyte             |   2|byte                                    |
|`0x19`     |writebyte             |   2|variable                                |
|`0x1a`     |writehalfword         |   3|halfword                                |
|`0x1b`     |writehalfword         |   2|variable                                |
|`0x1c`     |writeword             |   5|word                                    |
|`0x1d`     |writeword             |   2|variable                                |
|`0x1e`     |truncate              |   6|variable, word                          |
|`0x1f`     |truncate              |   3|variable, variable                      |
|`0x20`     |add                   |  10|variable, word, word                    |
|`0x21`     |add                   |   7|variable, word, variable                |
|`0x22`     |add                   |   7|variable, variable, word                |
|`0x23`     |add                   |   4|variable, variable, variable            |
|`0x24`     |subtract              |  10|variable, word, word                    |
|`0x25`     |subtract              |   7|variable, word, variable                |
|`0x26`     |subtract              |   7|variable, variable, word                |
|`0x27`     |subtract              |   4|variable, variable, variable            |
|`0x28`     |multiply              |  10|variable, word, word                    |
|`0x29`     |multiply              |   7|variable, word, variable                |
|`0x2a`     |multiply              |   7|variable, variable, word                |
|`0x2b`     |multiply              |   4|variable, variable, variable            |
|`0x2c`     |divide                |  10|variable, word, word                    |
|`0x2d`     |divide                |   7|variable, word, variable                |
|`0x2e`     |divide                |   7|variable, variable, word                |
|`0x2f`     |divide                |   4|variable, variable, variable            |
|`0x30`     |remainder             |  10|variable, word, word                    |
|`0x31`     |remainder             |   7|variable, word, variable                |
|`0x32`     |remainder             |   7|variable, variable, word                |
|`0x33`     |remainder             |   4|variable, variable, variable            |
|`0x34`     |and                   |  10|variable, word, word                    |
|`0x35`     |and                   |   7|variable, word, variable                |
|`0x36`     |and                   |   7|variable, variable, word                |
|`0x37`     |and                   |   4|variable, variable, variable            |
|`0x38`     |or                    |  10|variable, word, word                    |
|`0x39`     |or                    |   7|variable, word, variable                |
|`0x3a`     |or                    |   7|variable, variable, word                |
|`0x3b`     |or                    |   4|variable, variable, variable            |
|`0x3c`     |xor                   |  10|variable, word, word                    |
|`0x3d`     |xor                   |   7|variable, word, variable                |
|`0x3e`     |xor                   |   7|variable, variable, word                |
|`0x3f`     |xor                   |   4|variable, variable, variable            |
|`0x40`     |iflt                  |  10|variable, word, word                    |
|`0x41`     |iflt                  |   7|variable, word, variable                |
|`0x42`     |iflt                  |   7|variable, variable, word                |
|`0x43`     |iflt                  |   4|variable, variable, variable            |
|`0x44`     |ifle                  |  10|variable, word, word                    |
|`0x45`     |ifle                  |   7|variable, word, variable                |
|`0x46`     |ifle                  |   7|variable, variable, word                |
|`0x47`     |ifle                  |   4|variable, variable, variable            |
|`0x48`     |ifgt                  |  10|variable, word, word                    |
|`0x49`     |ifgt                  |   7|variable, word, variable                |
|`0x4a`     |ifgt                  |   7|variable, variable, word                |
|`0x4b`     |ifgt                  |   4|variable, variable, variable            |
|`0x4c`     |ifge                  |  10|variable, word, word                    |
|`0x4d`     |ifge                  |   7|variable, word, variable                |
|`0x4e`     |ifge                  |   7|variable, variable, word                |
|`0x4f`     |ifge                  |   4|variable, variable, variable            |
|`0x50`     |ifeq                  |  10|variable, word, word                    |
|`0x51`     |ifeq                  |   7|variable, word, variable                |
|`0x52`     |ifeq                  |   7|variable, variable, word                |
|`0x53`     |ifeq                  |   4|variable, variable, variable            |
|`0x54`     |ifne                  |  10|variable, word, word                    |
|`0x55`     |ifne                  |   7|variable, word, variable                |
|`0x56`     |ifne                  |   7|variable, variable, word                |
|`0x57`     |ifne                  |   4|variable, variable, variable            |
|`0x58`     |jumpz                 |   6|variable, word                          |
|`0x59`     |jumpz                 |   3|variable, variable                      |
|`0x5a`     |jumpnz                |   6|variable, word                          |
|`0x5b`     |jumpnz                |   3|variable, variable                      |
|`0x5c`     |callz                 |   6|variable, word                          |
|`0x5d`     |callz                 |   3|variable, variable                      |
|`0x5e`     |callnz                |   6|variable, word                          |
|`0x5f`     |callnz                |   3|variable, variable                      |
|`0x60`     |seek                  |   5|word                                    |
|`0x61`     |seek                  |   2|variable                                |
|`0x62`     |seekfwd               |   5|word                                    |
|`0x63`     |seekfwd               |   2|variable                                |
|`0x64`     |seekback              |   5|word                                    |
|`0x65`     |seekback              |   2|variable                                |
|`0x66`     |seekend               |   5|word                                    |
|`0x67`     |seekend               |   2|variable                                |
|`0x68`     |print                 |   5|word                                    |
|`0x69`     |print                 |   2|variable                                |
|`0x6a`     |menu                  |   6|variable, word                          |
|`0x6b`     |menu                  |   3|variable, variable                      |
|`0x6c`     |xordata               |   9|word, word                              |
|`0x6d`     |xordata               |   6|word, variable                          |
|`0x6e`     |xordata               |   6|variable, word                          |
|`0x6f`     |xordata               |   3|variable, variable                      |
|`0x70`     |fillbyte              |   6|word, byte                              |
|`0x71`     |fillbyte              |   6|word, variable                          |
|`0x72`     |fillbyte              |   3|variable, byte                          |
|`0x73`     |fillbyte              |   3|variable, variable                      |
|`0x74`     |fillhalfword          |   7|word, halfword                          |
|`0x75`     |fillhalfword          |   6|word, variable                          |
|`0x76`     |fillhalfword          |   4|variable, halfword                      |
|`0x77`     |fillhalfword          |   3|variable, variable                      |
|`0x78`     |fillword              |   9|word, word                              |
|`0x79`     |fillword              |   6|word, variable                          |
|`0x7a`     |fillword              |   6|variable, word                          |
|`0x7b`     |fillword              |   3|variable, variable                      |
|`0x7c`     |writedata             |   9|word, word                              |
|`0x7d`     |writedata             |   6|word, variable                          |
|`0x7e`     |writedata             |   6|variable, word                          |
|`0x7f`     |writedata             |   3|variable, variable                      |
|`0x80`     |lockpos               |   1|none                                    |
|`0x81`     |unlockpos             |   1|none                                    |
|`0x82`     |truncatepos           |   1|none                                    |
|`0x83`     |jumptable             |   2|variable                                |
|`0x84`     |set                   |   6|variable, word                          |
|`0x85`     |set                   |   3|variable, variable                      |
|`0x86`     |ipspatch              |   5|word                                    |
|`0x87`     |ipspatch              |   2|variable                                |
|`0x88`     |stackwrite            |   9|word, word                              |
|`0x89`     |stackwrite            |   6|word, variable                          |
|`0x8a`     |stackwrite            |   6|variable, word                          |
|`0x8b`     |stackwrite            |   3|variable, variable                      |
|`0x8c`     |stackread             |   6|variable, word                          |
|`0x8d`     |stackread             |   3|variable, variable                      |
|`0x8e`     |stackshift            |   5|word                                    |
|`0x8f`     |stackshift            |   2|variable                                |
|`0x90`     |retz                  |   2|variable                                |
|`0x91`     |retnz                 |   2|variable                                |
|`0x92`     |pushpos               |   1|none                                    |
|`0x93`     |poppos                |   1|none                                    |
|`0x94`     |bsppatch              |  10|variable, word, word                    |
|`0x95`     |bsppatch              |   7|variable, word, variable                |
|`0x96`     |bsppatch              |   7|variable, variable, word                |
|`0x97`     |bsppatch              |   4|variable, variable, variable            |
|`0x98`     |getbyteinc            |   3|variable, variable                      |
|`0x99`     |gethalfwordinc        |   3|variable, variable                      |
|`0x9a`     |getwordinc            |   3|variable, variable                      |
|`0x9b`     |increment             |   2|variable                                |
|`0x9c`     |getbytedec            |   3|variable, variable                      |
|`0x9d`     |gethalfworddec        |   3|variable, variable                      |
|`0x9e`     |getworddec            |   3|variable, variable                      |
|`0x9f`     |decrement             |   2|variable                                |

## Instruction set

The different instructions are listed here in alphabetical order, for lookup convenience. They are described in
separate sections.

* [add][calc]
* [and][calc]
* bsppatch
* [call][flow]
* [callnz][flow]
* [callz][flow]
* checksha1
* [decrement][var-basic]
* [divide][calc]
* [exit]
* fillbyte
* fillhalfword
* fillword
* getbyte
* getbytedec
* getbyteinc
* gethalfword
* gethalfworddec
* gethalfwordinc
* getword
* getworddec
* getwordinc
* [ifeq][conditionals]
* [ifge][conditionals]
* [ifgt][conditionals]
* [ifle][conditionals]
* [iflt][conditionals]
* [ifne][conditionals]
* [increment][var-basic]
* ipspatch
* [jump][flow]
* [jumpnz][flow]
* [jumptable]
* [jumpz][flow]
* length
* lockpos
* [menu]
* [multiply][calc]
* [nop]
* [or][calc]
* [pop][stack-basic]
* poppos
* pos
* [print]
* [push][stack-basic]
* pushpos
* readbyte
* readhalfword
* readword
* [remainder][calc]
* [retnz][flow]
* [return][flow]
* [retz][flow]
* seek
* seekback
* seekend
* seekfwd
* [set][var-basic]
* stackread
* stackshift
* stackwrite
* [subtract][calc]
* truncate
* truncatepos
* unlockpos
* writebyte
* writedata
* writehalfword
* writeword
* [xor][calc]
* xordata

[nop]: #no-operation
[var-basic]: #basic-variable-operations
[calc]: #arithmetical-and-logical-instructions
[stack-basic]: #basic-stack-operations
[flow]: #control-flow
[conditionals]: #conditionals
[exit]: #exiting
[print]: #printing-messages
[menu]: #option-menus
[jumptable]: #jump-tables

## Instruction description

Instructions are detailed here, including their operands and semantics.

For simplicity, the script compiler's syntax is used to show the form of the instruction. Operands that must be
variables are prefixed with `#`; other operands can be either variables or immediates. (In no case an instruction
only accepts an immediate as an operand.)

### No operation

```
nop
```

This instruction does nothing at all. It can be used, for instance, as a filler.

### Basic variable operations

```
set #variable, any
increment #variable
decrement #variable
```

The `set` instruction sets the variable's value to the specified value, which can be an immediate value or another
variable.

The `increment` and `decrement` respectively add and subtract one from the specified variable. While they are
equivalent to using `add #variable, #variable, 1` or `subtract #variable, #variable, 1`, they are available as shorter
forms (that also use fewer bytes in the BSP file).

### Arithmetical and logical instructions

```
add #variable, any, any
subtract #variable, any, any
multiply #variable, any, any
divide #variable, any, any
remainder #variable, any, any
and #variable, any, any
or #variable, any, any
xor #variable, any, any
```

These instructions perform the specified calculation between the two last operands and store the result in the variable
indicated by the first. The operands to the calculation are treated as unsigned values in all cases.

If the last operand to `divide` or `remainder` is zero, a fatal error occurs.

Note that the script compiler accepts a two-operand shorthand for these instructions, that is simply expanded to the
full three-operand form by repeating the first operand. (That is, `add #var, 3` is converted to `add #var, #var, 3`
prior to compilation.) This shorthand is a feature of the compiler, not part of the specification for the instructions;
the instructions (in the binary file) can only exist in three-operand form.

### Basic stack operations

```
push any
pop #variable
```

These instructions perform basic stack manipulations. The `push` instruction pushes a value into the stack (which can
be either a variable or a word immediate), and the `pop` instruction pops a value from the stack and stores it in the
specified variable.

If the `pop` instruction is executed with an empty stack, a fatal error occurs.

### Control flow

```
jump address
jumpz #variable, address
jumpnz #variable, address

call address
callz #variable, address
callnz #variable, address

return
retz #variable
retnz #variable
```

The `jump` instruction updates the instruction pointer, setting it to the value in its operand, which causes control to
jump to the instruction pointed by it. (Note that the address operands in these instructions can be immediate addresses
or variables.)

The `call` instruction does the same, but it first pushes the current value of the instruction pointer (which points to
the next instruction) to the stack. The `return` instruction pops a value from the stack and sets the instruction
pointer to it, thus returning from a prior call; executing `return` with an empty stack is equivalent to `exit 0`.

The `z` versions of the instructions above execute conditionally based on the value of a variable: they execute if the
variable is zero, or do nothing otherwise. The `nz` versions invert this condition.

### Conditionals

```
ifeq #variable, any, address
ifne #variable, any, address
iflt #variable, any, address
ifle #variable, any, address
ifgt #variable, any, address
ifge #variable, any, address
```

These instructions conditionally jump to the specified address, based on the condition specified by the instruction
itself. Respectively, the conditions are that the value is equal, not equal, less than, less than or equal, greater
than, and greater than or equal than the second argument to the instruction.

### Exiting

```
exit any
```

This instruction terminates the execution of the script. The operand to this instruction is the exit status of the BSP:
zero indicates success, and any other value indicates failure (the engine may use this value in any way it wishes as
long as it follows this convention). The engine must write out the contents of the file buffer to the output (the patch
target) upon exit with a status of zero.

If the current BSP is being executed as a result of a parent script executing a `bsppatch` instruction, the argument to
exit becomes the exit status that the parent `bsppatch` instruction will store. In this case, execution resumes with
the parent script; the engine must not terminate execution (regardless of exit status) and must not write to the output
(again, regardless of exit status) upon exit of a script invoked by `bsppatch`.

### Printing messages

```
print address
```

This instruction prints a message to the user. The address parameter indicates where in the patch space the message
begins; the message must be UTF-8 encoded, and continues until a null byte (a byte with value `0x00`) is found.

This document does not specify how the engine will display the message; however, in case of a console-based application
(or an environment that behaves in a similar fashion), it is recommended that the engine prints a newline character
after the message.

### Option menus

```
menu #variable, address
```

This instruction displays a menu with options, from which the user has to select one. The selected option number is
written to the indicated variable; the first option is numbered zero.

The address parameter should point to a list of pointers in patch space, each pointer pointing in turn to the text that
each option will contain (in the same format that the `print` instruction uses); the list is terminated with a
`0xffffffff` value. For instance, this code would display a menu with four options:

```
	menu #result, Options
	; ...

Options:
	dw .zero
	dw .one
	dw .two
	dw .three
	dw -1
.zero
	string "Option 0"
.one
	string "Option 1"
.two
	string "Option 2"
.three
	string "Option 3"
```

If the list of pointers is empty (i.e., if the first pointer is `0xffffffff`), no menu is shown to the user, and the
variable is set to `0xffffffff`.

### Jump tables

```
jumptable #variable
```

This instruction expects to be followed by a list of pointers, as many as needed according to the values the variable
may have. The instruction will read the word at `instruction pointer + 4 * #variable` and jump to the address that is
read. Note that no bounds checking is performed on the value of the variable, so the script must ensure that the
instruction will jump to a correct location.

For instance, the following code will jump to `.zero`, `.one` or `.two` depending on whether the value of `#var` is 0,
1 or 2 respectively (if `#var` is 3 or greater, the result is undefined):

```
	jumptable #var
	dw .zero
	dw .one
	dw .two

.zero
	; ...
.one
	; ...
.two
	; ...
```