---
layout: post
author: Jose Krause
title: "Quick intro to go assembly"
subtitle: "XOR AX, AX"
date: 2018-03-23T16:04:26+01:00
tags: ["golang", "asm", "macos"]
---


<!-- Code is in samuraiginger project -->

There is plenty of assembler in go's sourcebase, the best examples are on the `math/big`, `runtime` and `crypto` libraries, but beginning from zero with these is a little bit painful, the given examples are running code focused on system operations and performance.

This makes that going through this code for learning purposes can be some kinda hard for the unexperienced gopher. This is the reason for this brief article.

The Go ASM is an unusual form of assembly language used by the Go Compiler, and it is based on the Plan 9 input style, so it will be a good idea to read the [documentation](https://9p.io/sys/doc/asm.pdf) first.

-----

_**NOTE**: The target of this content is focused to x86\_64 architectures, but most of the examples are also valid for x86._
 
_Some of the examples are extracted from the original documentation with the goal in mind to create a comprehensive unified compendium of the most important/useful topics._

-----

<!-- ## First caveats -->

## First approach
Golang's ASM is a little bit different from the standard syntaxes like _NASM_ or _YASM_, one of the first things you come against is the fact that it is architecture independent, being no 32 or 64 bit register naming, as shown in the next example:

| NASM x86 | NASM x64 | Go ASM |
| -------- | -------- | ------ |
| eax      | rax      | AX     |
| ebx      | rbx      | BX     |
| ecx      | rcx      | CX     |
| …        | …        | …      |

The full set of register symbols depends on the architecture.

In addition, Go ASM has four predeclared symbols that refer to pseudo-registers. These are not real registers, but rather virtual registers maintained by the toolchain, the set is the same for all architectures:

* `FP`: Frame Pointer –Arguments and locals–
* `PC`: Program Counter –Jumps and branches–
* `SB`: Static Base pointer –Global symbols–
* `SP`: Stack Pointer –Top of the stack–.

These virtual registers are playing an important role in Go ASM, and are used constantly, the most important ones are `SB` and `FP`.

The pseudo-register `SB` can be thought as the origin of the memory, so the symbol `foo(SB)` is the name `foo` as an address in memory. This syntax has two basic modifiers, `<>` and `+N` where `N` is a integer. The first one `foo<>(SB)` means a _private_ element, only accessible from the same source file, like a lowercase name in Go, the second one is used to add an offset to the relative address that the name refers to, so `foo+8(SB)` is 8 bytes past the start of `foo`.

The `FP` pseudo-register is a _virtual Frame Pointer_ used to refer to procedure arguments, these references are maintained by the compiler referencing to the arguments on the stack as offsets from the pseudo-register. On a 64-bit machine, `0(FP)` is the first argument, `8(FP)` the second one and so on. To reference to this arguments, the compiler enforces the use of a name for it, this is for the sake of clarity and readability, so `MOVL foo+0(FP), CX` wil point the first argument from the virtual `FP` register to the physical `CX` register and with `MOVL bar+8(FP), DX` the second one to the `DX` register.

The reader will have already noticed that the ASM syntax is some kind of AT&T flavored, being not exactly the same:

| Intel              | AT&T                 | Go                 |
| ------------------ | -------------------- | ------------------ |
| `mov eax, 1`       | `movl $1, %eax`      | `MOVQ $1, AX`      |
| `mov rbx, 0ffh`    | `movl $0xff, %rbx`   | `MOVQ $(0xff), BX` |
| `mov ecx, [ebx+3]` | `movl 3(%ebx), %ecx` | `MOVQ 2(BX), CX`   |

Another significant difference is the global source-code file structure, on _NASM_ this structure is defined clearly with sections:

```asm
global start

section .bss
	…

section .data
	…

section .text
start:
	mov 	rax, 0x2000001
	mov 	rdi, 0x00
	syscall
```

On go asm this is done by preceding the symbol by a section type:

```go
DATA 	myInt<>+0x00(SB)/8, $42
GLOBL 	myInt<>(SB), RODATA, $8

// func foo()
TEXT ·foo(SB), NOSPLIT, $0
	MOVQ 	$0, DX
	LEAQ 	myInt<>(SB), DX
	RET
```

This syntax makes that defining symbols on the best place that makes sense for us is allowed.


## Calling ASM from Go code

As noted in the introduction, ASM in go is used for optimizations and low level system interactions, this makes that Go ASM is not intended to run standalone in the same form that classic ASM does, Go ASM must be called from Go source-code.

This might sound difficult to do, but like any other thing in Go doing this is very easy:

`hello.go`
```go
package main

func neg(x uint64) int64

func main() {
	println(neg(42))
}
```

`hello_amd64.s`
```go
TEXT ·neg(SB), NOSPLIT, $0
	MOVQ 	x+0(FP), AX
	NEGQ 	AX
	MOVQ 	AX, ret+8(FP)
	RET
```

Building and executing this will print a nice `-42` in our terminal.

Note the `·` unicode middle dot at the beginning of the procedure symbol, this is needed for package name separation, without anything preceding it `·foo` is the same as `main·foo`.

The procedure signature `TEXT ·neg(SB), NOSPLIT, $0` means:

* `TEXT`: The symbol is placed in the `text` section.
* `·neg`: The procedure's package symbol and symbol.
* `(SB)`: Needed by the lexer.
* `NOSPLIT`: Makes defining the arguments size unnecessary. –Could be omitted–
* `$0`: The size of the arguments, `$0` if `NOSPLIT` is defined

The build process is done as usual, `go build`, the GC will link the `.s` file automatically according to the suffix of the filename –`amd64`–.

As an extra resource to learn how a Go file is compiled we can take a look at the generated Go ASM with `go tool build -S <file.go>`.

Some symbols like `NOSPLIT` and `RODATA` are defined in the `textflax` header file, hence including this file with `#include textflag.h` is a good idea for a flawless compilation without strange errors.

## Syscalls on MacOS

Syscalls on MacOS are called adding the syscall number to `0x2000000`, for example, the exit syscall would be `0x2000001`, the reason of the `2` at the beginning of the syscall is that there are multiple classes of calls defined overlapping the call number, these types are defined [here](https://opensource.apple.com/source/xnu/xnu-792.10.96/osfmk/mach/i386/syscall_sw.h):

```c
#define SYSCALL_CLASS_NONE	0	/* Invalid */
#define SYSCALL_CLASS_MACH	1	/* Mach */	
#define SYSCALL_CLASS_UNIX	2	/* Unix/BSD */
#define SYSCALL_CLASS_MDEP	3	/* Machine-dependent */
#define SYSCALL_CLASS_DIAG	4	/* Diagnostics */
```

And a full list of MacOS syscall numbers can be found [here](https://opensource.apple.com/source/xnu/xnu-1504.3.12/bsd/kern/syscalls.master).

The arguments are passed to the syscall with the registers: `DI`, `SI`, `DX`, `R10`, `R8` and `R9`, the syscall code is stored in `AX`.

An example in _NASM_ would be:

```nasm
mov     rax, 0x2000004 		; write syscall
mov     rdi, 1 			; arg 1 fd (stdout)
mov     rsi, rcx 		; arg 2 buf
mov     rdx, 16 		; arg 3 count
syscall
```

In opposition, on Go ASM the same example would look like:

```go
MOVL 	$1,  DI 		// arg 1 fd (stdout)
LEAQ 	CX,  SI 		// arg 2 buf
MOVL 	$16, DX 		// arg 3 count
MOVL 	$(0x2000000+4), AX 	// syscall write
SYSCALL
```

By consensus, the syscall code is placed just before the `SYSCALL` instruction, this is purely community agreed, you could simply write the syscall at first like in nasm and it will compile without any problem.

## Using strings

Once at this point I'm sure you are capable of writing some basic ASM stuff and run it, for example the typical hello world, we know how to pass an argument to the procedure, how to return things and how to define a symbol on the data section. Have you tried to define a string?

I came against this problem a few days ago setting up some ASM code and my obvious question was, how do I define a fucking string?, ok, the classic approach of _NASM_ of defining the string like this:

```nasm
section data:
	foo: db "My random string", 0x00
```

Wasn't a valid option, after digging into every go ASM project I could find open on the Internet I didn't came against any example where a simple string definition where done. At the end the solution was to look at the Plan9 assembler documentation and with an example finding out how to bring this code to work.


The only difference to the Plan9 definition is the use of double quotes instead of single ones and add the symbol `RODATA`:

```go
DATA  foo<>+0x00(SB)/8, $"My rando"
DATA  foo<>+0x08(SB)/8, $"m string"
DATA  foo<>+0x16(SB)/1, $0x0a
GLOBL foo<>(SB), RODATA, $24

TEXT ·helloWorld(SB), NOSPLIT, $0
	MOVL 	$(0x2000000+4), AX 	// syscall write
	MOVQ 	$1, DI 			// arg 1 fd
	LEAQ 	foo<>(SB), SI 		// arg 2 buf
	MOVL 	$24, DX 		// arg 3 count
	SYSCALL
	RET
```

Note that the string definition couldn't be made at once, there is needed to define it in chunks of 8 bytes –64 bits–

Now you are ready to drive deeper into the Go ASM world writing your own super-fast and ultra-optimized code, and remember, RTFM ;)

## Uses in security?

Aside from code optimizations using ASM on your go code is really handy for Antivirus Software evasion avoiding to trigger well known signatures, using some useful anti-debugging techniques to evade sandboxes looking for abnormal behavior or make cry an analyst just for fun.

If you're interested on this, I will write about this topic in a next post, stay tuned!


## Additional links
* [https://golang.org/doc/asm](https://golang.org/doc/asm)
* [https://9p.io/sys/doc/asm.pdf](https://9p.io/sys/doc/asm.pdf)
* [https://goroutines.com/asm](https://goroutines.com/asm)
* [https://blog.sgmansfield.com/2017/04/a-foray-into-go-assembly-programming/](https://blog.sgmansfield.com/2017/04/a-foray-into-go-assembly-programming/)