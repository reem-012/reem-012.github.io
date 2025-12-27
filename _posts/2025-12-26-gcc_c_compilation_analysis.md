---
layout: post
title: "GCC C Compilation Analysis"
date: 2025-12-26
---
# GCC Compilation Analysis

This document will be going over how a program goes from source code to a binary that you can run. All examples are performed on `Ubuntu 24.04.3 LTS`. 

Please Note that there may be errors or incorrect explainations in this document so do not treat this as gospel as I was learning this for fun

## The Four Stages of Compilation

When you compile a program it does not go straight from a .c to a binary. It goes through a 4 phase process

1. Preprocessing (.c → .i): Handles #include, #define, #ifdef, etc.
2. Compilation (.i → .s): Translates C code to assembly language
3. Assembly (.s → .o): Converts assembly to machine code (object file)
4. Linking (.o → executable): Combines object files and libraries

## Command
```bash
gcc -save-temps -v hello.c -o hello
```

This runs GCC in verbose mode and saves intermediate files:
- `hello` - final executable
- `hello.i` - preprocessed source
- `hello.s` - assembly code
- `hello.o` - object file

## Analyzing the Preprocessing Stage

### What is the Preprocessing Stage For

The preprocessor handles all directives starting with #:

1. Expands #include - Copies entire header files into your code
2. Expands macros - Replaces #define macros with their values
3. Conditional compilation - Processes #ifdef, #ifndef, #if
4. Removes comments - Strips out // and /* */ comments

To make this a little bit clearer, we will take a program and call it `max.c`

```c
#define MAX 100
#include <stdio.h>

int main() {
    // This is a comment
    printf("Max is %d\n", MAX);
    return 0;
}
```

Let's go ahead and preprocess this 

```bash
$ gcc -E max.c -o max.i
$ tail max.i

# 3 "max.c" 2


# 4 "max.c"
int main() {

    printf("Max is %d\n", 100);
    return 0;
}
```

This is going to create an intermediate file `max.i`.

As you can see, what was originally line 6 in our program now has the macro MAX substituted for its actual value 

```c
printf("Max is %d\n", 100);
```

### How Preprocessing Works with hello.c

If we examine the output of the verbose gcc, we can see that gcc is calling cc1 to do its preprocessing. `cc1` is actually the C compiler proper (the core compiler component), and the `-E` flag tells it to stop after preprocessing. It's not a separate preprocessor program.

```bash
/usr/libexec/gcc/x86_64-linux-gnu/13/cc1 -E -quiet -v -imultiarch x86_64-linux-gnu hello.c -mtune=generic -march=x86-64 -fpch-preprocess -fasynchronous-unwind-tables -fstack-protector-strong -Wformat -Wformat-security -fstack-clash-protection -fcf-protection -o hello.i
```

This creates `hello.i`, which is massive compared to our original source. The reason for this is because the preprocessing is literally copying the entire contents of stdio.h (and all of the includes that stdio uses) and is putting them into the intermediate file. The preprocessor expanded `#include <stdio.h>`. 

```bash
$ wc hello.i
  819  2358 21312 hello.i
```

### Line Directives for Error Tracking

Line directives tell the compiler where code originated, enabling accurate error reporting. At the end of `hello.i`:
```c
# 3 "hello.c"
int main() {
    printf("Hello, World!\n");
    return 0;
}
```
This indicates errors here should reference line 3 of `hello.c`.

Similarly, for library functions:
```c
# 184 "/usr/include/stdio.h" 3 4
extern int fclose (FILE *__stream) __attribute__ ((__nonnull__ (1)));
```
Any errors with `fclose` will point to line 184 of `stdio.h`.

This creates a complete "breadcrumb trail" so the compiler always knows the original source location of any line of code, no matter how deeply nested the includes are.

If you notice that there are sometimes flags after a line directive, these have various meanings: 

- 1 = start of a new file (entering an include)
- 2 = returning from an include file
- 3 = following text comes from system header
- 4 = following text should be treated as wrapped in extern "C"

### Cool Trick: Compiling Intermediate Code

So we have already gone through and explained how intermediate code is basically the output of the preprocessor doing its work to get everything ready for compilation. The preprocessor does generate fully valid C code though. You can manually compile and run the intermediate code that is generated. There is no real reason to ever do this, but it is possible for you to do

```bash
$ gcc hello.i -o intermediate_hello
$ chmod u+x intermediate_hello 
$ ./intermediate_hello 
Hello, World! 
```
## Analyzing the Compilation Phase

Looking at the GCC verbose output, we can see the following line:

```bash
/usr/libexec/gcc/x86_64-linux-gnu/13/cc1 -fpreprocessed hello.i -quiet -dumpbase hello.c -dumpbase-ext .c -mtune=generic -march=x86-64 -version -fasynchronous-unwind-tables -fstack-protector-strong -Wformat -Wformat-security -fstack-clash-protection -fcf-protection -o hello.s
```

This shows GCC utilizing `cc1` to take our preprocessed `hello.i` file and create an assembly file `hello.s`.

When we examine `hello.s`, we can see that it contains an assembly representation of our `hello.c` program.

### Optimization Levels

So one of the interesting things to explore when it comes to examining the optimization that GCC does when compiling your intermediate file to assembly. If you read [this](https://gcc.gnu.org/onlinedocs/gnat_ugn/Optimization-Levels.html) article you can see `Without any optimization switch, the compiler's goal is to reduce the cost of compilation and to make debugging produce the expected results`. 

There are various optimization flags that can be added to GCC. These flags will eventually get passed to cc1 when doing the assembly of the intermediate files. 

| Flag | Description |
|------|-------------|
| `-O0` | No optimization (the default); generates unoptimized code but has the fastest compilation time |
| `-O1` | Moderate optimization; optimizes reasonably well but does not degrade compilation time significantly |
| `-O2` | Extensive optimization; generates highly optimized code but has an increased compilation time |
| `-O3` | Full optimization; attempts more sophisticated transformations, possibly at the cost of larger generated code |
| `-Os` | Optimize for size of resulting binary rather than speed; based on `-O2` but reduces code size |
| `-Oz` | Optimize aggressively for size of resulting binary rather than speed |
| `-Og` | Optimize for debugging experience rather than speed |
| `-Ofast` | Optimize for speed without strict standards compliance; enables `-O3` plus additional optimizations |

We won't go too much into depth about how each optimization is working underneath the hood but you can definitely see the line difference.
```bash
$ wc -l hello_O*.s
  45 hello_O0.s
  41 hello_O1.s
  42 hello_O2.s
  42 hello_O3.s
  42 hello_Ofast.s
  41 hello_Og.s
  41 hello_Os.s
  41 hello_Oz.s

$ sha256sum hello_O*.s
a2835c52c20669a8d779d1c8fbee4fa06c2c36e40c0c9ff6f4ad75e3040337a0  hello_O0.s
2669564cd28aac9fa0ce25c3a7eea816dbdc900346cc10ffe2e6e0a76a1ecf78  hello_O1.s
75c8896a12904247624aeb82537056ec9ba6031bce7ddc449d1b8252e953f1d4  hello_O2.s
75c8896a12904247624aeb82537056ec9ba6031bce7ddc449d1b8252e953f1d4  hello_O3.s
75c8896a12904247624aeb82537056ec9ba6031bce7ddc449d1b8252e953f1d4  hello_Ofast.s
2669564cd28aac9fa0ce25c3a7eea816dbdc900346cc10ffe2e6e0a76a1ecf78  hello_Og.s
f4d57f00651384edc109f0b760f101b47b33cf142efedca33b6f4ce622a18219  hello_Os.s
f4d57f00651384edc109f0b760f101b47b33cf142efedca33b6f4ce622a18219  hello_Oz.s
```

Note that line count doesn't always directly correlate with performance; this is just to prove in a very easy way that there are differences between assembly files. I have also included the SHA hashes to further illustrate the point that the files are different. I implore you to run this on your own and see the differences yourself.

## Analyzing the Assembly Stage

After the assembly file (hello.s) it is now time to create our object file (hello.o). The object file is a machine code representation of the assembly file. Note that it is only in machine code; it is not an executable binary, meaning that you can't run the hello.o file.

If you look at the verbose output you will see the line `as -v --64 -o hello.o hello.s`. This is GCC using the GNU Assembler to convert hello.s to hello.o

### What is an object file

If you run the `file` command on `hello.o` you will see that it is a relocatable ELF file. This means that while it is a binary, it has not been linked yet. Compare that the the fully exectable binary and you will see the difference. 
```bash
$ # Get the file type of the Object File
$ file hello.o
hello.o: ELF 64-bit LSB relocatable, x86-64, version 1 (SYSV), not stripped
$ # Get the file type of the Exectuable File
$ file hello
hello: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=832594bbec3cdd9992fe40755f43ad6e4d7c11b8, for GNU/Linux 3.2.0, not stripped
```

So let's explore what that means a little bit more for an object to be unlinked. If we take a look at hello.o in a little bit more detail and run `objdump -h`, we can see the header files of our object file.

```bash
reem@linodedevelopment:~/dfir_roadmap/working/c_week1/hello_world$ objdump -h hello.o

hello.o:     file format elf64-x86-64

Sections:
Idx Name          Size      VMA               LMA               File off  Algn
  0 .text         0000001e  0000000000000000  0000000000000000  00000040  2**0
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, CODE
  1 .data         00000000  0000000000000000  0000000000000000  0000005e  2**0
                  CONTENTS, ALLOC, LOAD, DATA
  2 .bss          00000000  0000000000000000  0000000000000000  0000005e  2**0
                  ALLOC
  3 .rodata       0000000e  0000000000000000  0000000000000000  0000005e  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  4 .comment      0000002c  0000000000000000  0000000000000000  0000006c  2**0
                  CONTENTS, READONLY
  5 .note.GNU-stack 00000000  0000000000000000  0000000000000000  00000098  2**0
                  CONTENTS, READONLY
  6 .note.gnu.property 00000020  0000000000000000  0000000000000000  00000098  2**3
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  7 .eh_frame     00000038  0000000000000000  0000000000000000  000000b8  2**3
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, DATA
reem@linodedevelopment:~/dfir_roadmap/working/c_week1/hello_world$ 
```
| Column | Meaning |
|--------|---------|
| Idx | Section index number |
| Name | Section name |
| Size | Size of the section in hexadecimal bytes |
| VMA | Virtual Memory Address (where it will be loaded in memory) |
| LMA | Load Memory Address (where it's loaded from) |
| File off | Offset in the file where this section's data starts |
| Algn | Alignment requirement (2^n bytes) |

If you notice, the VMA of all the different header sections is 0000000000000000. This is because the object has not been linked yet. Right now everything is sort of like a puzzle piece; everything exists but it can be moved around. It is the job of the linker to take this object file and map the memory addresses.

## Analyzing the Linking Stage

In order to make our object file we have to run the program through the linker. If we take a look at the GCC output we see the line that is used to initate the linking process

```bash
 /usr/libexec/gcc/x86_64-linux-gnu/13/collect2 -plugin /usr/libexec/gcc/x86_64-linux-gnu/13/liblto_plugin.so -plugin-opt=/usr/libexec/gcc/x86_64-linux-gnu/13/lto-wrapper -plugin-opt=-fresolution=hello.res -plugin-opt=-pass-through=-lgcc -plugin-opt=-pass-through=-lgcc_s -plugin-opt=-pass-through=-lc -plugin-opt=-pass-through=-lgcc -plugin-opt=-pass-through=-lgcc_s --build-id --eh-frame-hdr -m elf_x86_64 --hash-style=gnu --as-needed -dynamic-linker /lib64/ld-linux-x86-64.so.2 -pie -z now -z relro -o hello /usr/lib/gcc/x86_64-linux-gnu/13/../../../x86_64-linux-gnu/Scrt1.o /usr/lib/gcc/x86_64-linux-gnu/13/../../../x86_64-linux-gnu/crti.o /usr/lib/gcc/x86_64-linux-gnu/13/crtbeginS.o -L/usr/lib/gcc/x86_64-linux-gnu/13 -L/usr/lib/gcc/x86_64-linux-gnu/13/../../../x86_64-linux-gnu -L/usr/lib/gcc/x86_64-linux-gnu/13/../../../../lib -L/lib/x86_64-linux-gnu -L/lib/../lib -L/usr/lib/x86_64-linux-gnu -L/usr/lib/../lib -L/usr/lib/gcc/x86_64-linux-gnu/13/../../.. hello.o -lgcc --push-state --as-needed -lgcc_s --pop-state -lc -lgcc --push-state --as-needed -lgcc_s --pop-state /usr/lib/gcc/x86_64-linux-gnu/13/crtendS.o /usr/lib/gcc/x86_64-linux-gnu/13/../../../x86_64-linux-gnu/crtn.o
```

If we use objdump to take a look at the headers now of `hello` we will see that there are actual VMA's set and not just 00's. As you can notice there is considerably more sections here. The reason for this is because the linker has expanded the program to be a fully functional executable. 
```bash
$ objdump -h hello

hello:     file format elf64-x86-64

Sections:
Idx Name          Size      VMA               LMA               File off  Algn
  0 .interp       0000001c  0000000000000318  0000000000000318  00000318  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  1 .note.gnu.property 00000030  0000000000000338  0000000000000338  00000338  2**3
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  2 .note.gnu.build-id 00000024  0000000000000368  0000000000000368  00000368  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  3 .note.ABI-tag 00000020  000000000000038c  000000000000038c  0000038c  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  4 .gnu.hash     00000024  00000000000003b0  00000000000003b0  000003b0  2**3
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  5 .dynsym       000000a8  00000000000003d8  00000000000003d8  000003d8  2**3
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  6 .dynstr       0000008d  0000000000000480  0000000000000480  00000480  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  7 .gnu.version  0000000e  000000000000050e  000000000000050e  0000050e  2**1
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  8 .gnu.version_r 00000030  0000000000000520  0000000000000520  00000520  2**3
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  9 .rela.dyn     000000c0  0000000000000550  0000000000000550  00000550  2**3
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
 10 .rela.plt     00000018  0000000000000610  0000000000000610  00000610  2**3
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
 11 .init         0000001b  0000000000001000  0000000000001000  00001000  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
 12 .plt          00000020  0000000000001020  0000000000001020  00001020  2**4
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
 13 .plt.got      00000010  0000000000001040  0000000000001040  00001040  2**4
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
 14 .plt.sec      00000010  0000000000001050  0000000000001050  00001050  2**4
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
 15 .text         00000107  0000000000001060  0000000000001060  00001060  2**4
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
 16 .fini         0000000d  0000000000001168  0000000000001168  00001168  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
 17 .rodata       00000012  0000000000002000  0000000000002000  00002000  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
 18 .eh_frame_hdr 00000034  0000000000002014  0000000000002014  00002014  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
 19 .eh_frame     000000ac  0000000000002048  0000000000002048  00002048  2**3
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
 20 .init_array   00000008  0000000000003db8  0000000000003db8  00002db8  2**3
                  CONTENTS, ALLOC, LOAD, DATA
 21 .fini_array   00000008  0000000000003dc0  0000000000003dc0  00002dc0  2**3
                  CONTENTS, ALLOC, LOAD, DATA
 22 .dynamic      000001f0  0000000000003dc8  0000000000003dc8  00002dc8  2**3
                  CONTENTS, ALLOC, LOAD, DATA
 23 .got          00000048  0000000000003fb8  0000000000003fb8  00002fb8  2**3
                  CONTENTS, ALLOC, LOAD, DATA
 24 .data         00000010  0000000000004000  0000000000004000  00003000  2**3
                  CONTENTS, ALLOC, LOAD, DATA
 25 .bss          00000008  0000000000004010  0000000000004010  00003010  2**0
                  ALLOC
 26 .comment      0000002b  0000000000000000  0000000000000000  00003010  2**0
                  CONTENTS, READONLY
```

Alot of the extra stuff that has been added is the required sections to enable dynamic linking. We can see the symblos that will dynamically be resolved at runtime with objdump 

```bash
$ objdump -T hello

hello:     file format elf64-x86-64

DYNAMIC SYMBOL TABLE:
0000000000000000      DF *UND*  0000000000000000 (GLIBC_2.34) __libc_start_main
0000000000000000  w   D  *UND*  0000000000000000  Base        _ITM_deregisterTMCloneTable
0000000000000000      DF *UND*  0000000000000000 (GLIBC_2.2.5) puts
0000000000000000  w   D  *UND*  0000000000000000  Base        __gmon_start__
0000000000000000  w   D  *UND*  0000000000000000  Base        _ITM_registerTMCloneTable
0000000000000000  w   DF *UND*  0000000000000000 (GLIBC_2.2.5) __cxa_finalize
```

You can see what libraries are needed using the readelf command

```bash
$ readelf -d hello | grep NEEDED 
 0x0000000000000001 (NEEDED)             Shared library: [libc.so.6]
```