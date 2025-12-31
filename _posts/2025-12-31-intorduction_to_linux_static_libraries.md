---
layout: post
title: "Introduction to Static Libraries in Linux"
date: 2025-12-31
---
# Introduction to Static Libraries in Linux
> I'm not a systems engineer—I study this as a hobby. If you spot any errors, feel free to reach out. All examples were done on Ubuntu 25.04 but should work on other distros and versions of Linux.

## Creating a Static Library

The steps to create a static library are relatively simple, and we will explore each step.

Creating the static library:

1. Create the implementation file. This is where the actual source code for your library lives.
2. Compile your implementation to an object file.
3. Create an archive for your library.

Using the static library:

4. Create a header file.
5. Link your program against the archive to produce an executable.

### Implementation File

To start, we are going to need to create an implementation file. This is going to be the source code of our library.

> Create a file `mymath.c`

```c
// mymath.c
int add(int a, int b) {
    return a + b;
}
```

As you can see, this is a very simple file that contains no `main()` function. This is because this is the code for a library that will be used by other programs. Libraries don't have entry points. They just provide code that other programs (which do have `main()`) can call.

This library is going to offer an `add()` function that other programs will be able to consume.

### Compiling Implementation

If we run `gcc -c mymath.c -o mymath.o`, this is going to create an object file (`mymath.o`).

If we run `file` on this, we can see that the file type is `ELF 64-bit LSB relocatable`:

```bash
$ file mymath.o

mymath.o: ELF 64-bit LSB relocatable, x86-64, version 1 (SYSV), not stripped
```



An `ELF 64-bit LSB relocatable` file is compiled machine code, but not yet an executable. It contains code with placeholder addresses since the compiler doesn't know where things will end up in memory yet. The linker resolves these placeholders later when building the final executable. The [ELF Documentation](https://refspecs.linuxfoundation.org/elf/elf.pdf) describes a relocatable files as follows

> relocatable files must have information that describes how to modify their section contents, thus allowing executable and shared object files to hold the right information for a process's program image. Relocation entries are these data.


So we know that this relocatable file is an object file. An object file is the output of a compiler before it has been linked. What that effectively means is that all the symbols don't have their final memory addresses yet. We can take a look at the symbol table like this:

```bash
$ readelf -s mymath.o

Symbol table '.symtab' contains 4 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS mymath.c
     2: 0000000000000000     0 SECTION LOCAL  DEFAULT    1 .text
     3: 0000000000000000    24 FUNC    GLOBAL DEFAULT    1 add
```

What each column of `readelf -s` means:

| Column | Meaning |
|--------|---------|
| Num | Index of the symbol in the table |
| Value | Memory address (all zeros until linked) |
| Size | Size of the symbol in bytes |
| Type | What the symbol is: `FUNC` (function), `FILE` (source file), `SECTION` (section marker), `NOTYPE` (undefined/unknown) |
| Bind | Visibility scope: `LOCAL` (only this file), `GLOBAL` (visible to other files during linking) |
| Vis | Symbol visibility for dynamic linking: `DEFAULT`, `HIDDEN`, `PROTECTED` |
| Ndx | Section index where the symbol lives: `1` (.text), `ABS` (absolute/no section), `UND` (undefined/external) |
| Name | The symbol's name |

We can see that all the addresses start at `0x0`. The compiler has compiled everything to machine code, but it doesn't know where the various components of the program will live in memory. This makes sense because this is a library.

At this point in the process, we can't assign an address to our `add` function since we have no context about how this function will be used in an actual program. We are effectively creating code that the linker will eventually reference when creating an executable.

### Creating the Archive

So now that we have created a compiled version of our library (mymath.o), we now need to create an archive out of it.

The man pages describe an archive as follows:

> An archive is a single file holding a collection of other files in a structure that makes it possible to retrieve the original individual files (called members of the archive).

Effectively, what you are doing is putting the object file into a bundle that can be used by a linker when creating an executable. The archive bundles the entire object file, including its symbol table, so the linker can see what functions are available.

To create the archive from our object file, we can run this command: `ar rcs libmymath.a mymath.o`. This is going to generate a file `libmymath.a`. The `ar` program creates, modifies, and extracts from archives. We included the flags `rcs` for the `ar` command. This is what each one does:

- `r` — Insert files into the archive
- `c` — Create the archive if it doesn't exist
- `s` — Write an index of symbols (this is effectively a higher-level index that speeds up the linker)

If we run `file` on it, we can see that this is an archive file:

```bash
$ file libmymath.a

libmymath.a: current ar archive
```

If we run the `readelf` command on this archive, we can see that the `mymath.o` file is referenced in here as well as its symbol table.

```bash
$ readelf -s libmymath.a

File: libmymath.a(mymath.o)

Symbol table '.symtab' contains 4 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS mymath.c
     2: 0000000000000000     0 SECTION LOCAL  DEFAULT    1 .text
     3: 0000000000000000    24 FUNC    GLOBAL DEFAULT    1 add
```

#### Analyzing Static Table
```bash
$ objdump -t libmymath.a 
In archive libmymath.a:

mymath.o:     file format elf64-x86-64

SYMBOL TABLE:
0000000000000000 l    df *ABS*  0000000000000000 mymath.c
0000000000000000 l    d  .text  0000000000000000 .text
0000000000000000 g     F .text  0000000000000018 add
```

## Using the Static Library
### Header File

Now that we have our archive created, we are actually able to use this library. The first thing that we are going to need to do to be able to use this library is create a header file.

> Create a file `mymath.h`
```c
// mymath.h
int add(int a, int b);
```

A header file basically just contains the shape of functions offered by a library. If we take a look at the example header file that we will be creating below, we can see that we don't explicitly state how the function is implemented. Rather, it is more like a contract for the compiler. By creating this file, we are also promising the compiler that even if we don't implement that function in our `main.c` file, the linker will be able to resolve it.

We will use this file in an include statement when we are creating our actual program.

### Create our Executable 

To create a program that uses this as a library, we first need to write the code for it. We will create a simple program that utilizes the `add()` function that we implemented in our library.

> Create a file `main.c`
```c
// main.c
#include <stdio.h>
#include "mymath.h"

int main() {
    printf("Result: %d\n", add(5, 5));
    return 0;
}
```

The main thing to note here is that we are including the header file that we created in the previous step. This allows us to tell the program how we expect the `add()` function to work. The reason that we do this is so we can compile the executable before linking. While the program is compiling, we have effectively promised the compiler that this code will be able to be linked by the linker and that it will function in a certain way.

To create our executable, we can run the command `gcc main.c -L. -lmymath -static -o static_prog`. This is going to create an executable called `static_prog`.

It can be run as follows:
```bash
$ ./static_prog 
Result: 10
```

If we run `file` on this, we can see that it is a statically linked executable.
```bash
$ file static_prog

static_prog: ELF 64-bit LSB executable, x86-64, version 1 (GNU/Linux), statically linked, BuildID[sha1]=115e285fadb9514b65145c5d6521cf4b2ac45779, for GNU/Linux 3.2.0, not stripped
```

If we take a look again at the GCC command we ran to compile this, we can break down all the sections:

- `main.c` — The name of the source code file
- `-L.` — Tells the linker to add the current directory to the list of libraries that it will search through
- `-lmymath` — Tells the linker to reference the archive we created. It effectively is looking for a file titled `libmymath.a`
- `-static` — Tells GCC to statically link everything. While this doesn't change anything for our static library, we also will be statically linking the libc shared library, which drastically increases the size which may not be desirable in all situations. We are doing this so we can only focus on static linking opposed to dynamic linking
- `-o static_prog` — Specifies the name of the output file

If we take a look at the disassembly for our object file, we can see the assembly for the `add` function. We can see that the `add()` function has an address of `0x0`. As we discussed earlier, this is because that object file is functioning as a relocatable binary and is supposed to be referenced by a linker.
```bash
$ objdump -d mymath.o

mymath.o:     file format elf64-x86-64


Disassembly of section .text:

0000000000000000 <add>:
   0:   f3 0f 1e fa             endbr64
   4:   55                      push   %rbp
   5:   48 89 e5                mov    %rsp,%rbp
   8:   89 7d fc                mov    %edi,-0x4(%rbp)
   b:   89 75 f8                mov    %esi,-0x8(%rbp)
   e:   8b 55 fc                mov    -0x4(%rbp),%edx
  11:   8b 45 f8                mov    -0x8(%rbp),%eax
  14:   01 d0                   add    %edx,%eax
  16:   5d                      pop    %rbp
  17:   c3                      ret
```

If we take a look at how the `add` function is implemented in our final executable, we can see that the assembly instructions are roughly the same (with a little bit of padding at the end that was added during linking for memory alignment). One thing to note is that since this code was linked into the final executable, the `add()` function now actually has a real memory address (`0x401839`).
```bash
$ objdump -d static_prog | grep -A 15 "<add>:"
0000000000401839 <add>:
  401839:       f3 0f 1e fa             endbr64
  40183d:       55                      push   %rbp
  40183e:       48 89 e5                mov    %rsp,%rbp
  401841:       89 7d fc                mov    %edi,-0x4(%rbp)
  401844:       89 75 f8                mov    %esi,-0x8(%rbp)
  401847:       8b 55 fc                mov    -0x4(%rbp),%edx
  40184a:       8b 45 f8                mov    -0x8(%rbp),%eax
  40184d:       01 d0                   add    %edx,%eax
  40184f:       5d                      pop    %rbp
  401850:       c3                      ret
  401851:       66 2e 0f 1f 84 00 00    cs nopw 0x0(%rax,%rax,1)
  401858:       00 00 00 
  40185b:       0f 1f 44 00 00          nopl   0x0(%rax,%rax,1)
```