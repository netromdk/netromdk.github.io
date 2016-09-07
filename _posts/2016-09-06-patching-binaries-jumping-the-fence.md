---
layout: post
title:  "Patching Binaries: Jumping The Fence"
date:   2016-09-06 21:43:55 +0200
tags:   patching x86 c++ python lldb disassembling
---

_Patching Binaries_ is a series of articles about how to extract information and modify program behavior. It focuses on the Mac [Mach-O][mach-o] executable format for the [x86][x86] architecture, but the techniques are similar for other formats.

Before proceeding it is recommended reading the [first part]({% post_url 2016-09-05-patching-binaries-strings %}) about the fundamentals of finding and changing constant strings in binaries.

_The files used in this article can be found [here](https://github.com/netromdk/patching/tree/master/jumps/)._

An approach to change the behavior of a program is to jump or bypass instructions. Or in other words: _jumping the fence!_

Compile and strip example program "verify.cc" as "verify" and run it:

```shell
% ./verify
Serial number: 83
Incorrect!

% ./verify
Serial number: 90
Correct!
```

If we disregard knowing from the source code how the numbers are verified, we would like to somehow bypass the check so that all inputs are deemed correct. There are numerous ways of doing this but the easiest is putting _NOP_ (No Operation - opcode `0x90`) instructions at tactical places in the program. They are a means of masking out functionality because they simply do nothing.

First we need to find out where to place the NOPs. And as a preliminary deviation we will look at how to determine a program's entry point address; where the first program instruction resides. When dealing with stripped executables we can't break at `main` anymore.

In Mach-O binaries the program instructions are located in section `__text` of segment `__TEXT`:

```shell
% otool -s __TEXT __text verify | head -n 3
verify:
Contents of (__TEXT,__text) section
00000001000010c0	55 48 89 e5 48 83 ec 30 48 8b 3d 39 0f 00 00 48
```

To easily retrieve the entry point and start [LLDB][lldb] with a break point on that address, take a look at "lldb_entrypoint.sh":

```sh
#!/bin/sh
ADDR=`otool -s __TEXT __text $1 | head -n 3 | tail -n 1 | awk '{print $1;}'`
echo "b 0x${ADDR}" > /tmp/.lldbcmds
lldb -s /tmp/.lldbcmds $1
```

With that let's start up the debugger with our program and reach the break point:

```shell
% ./lldb_entrypoint.sh verify
(lldb) target create "verify"
Current executable set to 'verify' (x86_64).
(lldb) command source -s 0 '/tmp/.lldbcmds'
Executing commands in '/tmp/.lldbcmds'.
(lldb) b 0x00000001000010c0
Breakpoint 1: address = 0x00000001000010c0
(lldb) r
Process 88913 launched: '/Users/netrom/git/patching/jumps/verify' (x86_64)
Process 88913 stopped
* thread #1: tid = 0x233953, 0x00000001000010c0 verify`___lldb_unnamed_symbol1$$verify, queue = 'com.apple.main-thread', stop reason = breakpoint 1.1
    frame #0: 0x00000001000010c0 verify`___lldb_unnamed_symbol1$$verify
verify`___lldb_unnamed_symbol1$$verify:
->  0x1000010c0 <+0>: pushq  %rbp
    0x1000010c1 <+1>: movq   %rsp, %rbp
    0x1000010c4 <+4>: subq   $0x30, %rsp
    0x1000010c8 <+8>: movq   0xf39(%rip), %rdi         ; (void *)0x00007fff78ba32f8: std::__1::cout
(lldb)
```

The next step is to disassemble to get an overview:

```shell
(lldb) dis
verify`___lldb_unnamed_symbol1$$verify:
->  0x1000010c0 <+0>:   pushq  %rbp
    0x1000010c1 <+1>:   movq   %rsp, %rbp
    0x1000010c4 <+4>:   subq   $0x30, %rsp
    0x1000010c8 <+8>:   movq   0xf39(%rip), %rdi         ; (void *)0x00007fff78ba32f8: std::__1::cout
    0x1000010cf <+15>:  leaq   0xe86(%rip), %rsi         ; "Serial number: "
    0x1000010d6 <+22>:  movl   $0x0, -0x4(%rbp)
    0x1000010dd <+29>:  callq  0x100001150               ; ___lldb_unnamed_symbol2$$verify
    0x1000010e2 <+34>:  movq   0xf17(%rip), %rdi         ; (void *)0x00007fff78ba3250: std::__1::cin
    0x1000010e9 <+41>:  leaq   -0x8(%rbp), %rsi
    0x1000010ed <+45>:  movq   %rax, -0x10(%rbp)
    0x1000010f1 <+49>:  callq  0x100001d74               ; symbol stub for: std::__1::basic_istream<char, std::__1::char_traits<char> >::operator>>(int&)
    0x1000010f6 <+54>:  movl   -0x8(%rbp), %edi
    0x1000010f9 <+57>:  movq   %rax, -0x18(%rbp)
    0x1000010fd <+61>:  callq  0x1000011a0               ; ___lldb_unnamed_symbol3$$verify
    0x100001102 <+66>:  testb  $0x1, %al
    0x100001104 <+68>:  jne    0x10000110f               ; <+79>
    0x10000110a <+74>:  jmp    0x10000112b               ; <+107>
    0x10000110f <+79>:  movq   0xef2(%rip), %rdi         ; (void *)0x00007fff78ba32f8: std::__1::cout
    0x100001116 <+86>:  leaq   0xe4f(%rip), %rsi         ; "Correct!\n"
    0x10000111d <+93>:  callq  0x100001150               ; ___lldb_unnamed_symbol2$$verify
    0x100001122 <+98>:  movq   %rax, -0x20(%rbp)
    0x100001126 <+102>: jmp    0x100001142               ; <+130>
    0x10000112b <+107>: movq   0xed6(%rip), %rdi         ; (void *)0x00007fff78ba32f8: std::__1::cout
    0x100001132 <+114>: leaq   0xe3d(%rip), %rsi         ; "Incorrect!\n"
    0x100001139 <+121>: callq  0x100001150               ; ___lldb_unnamed_symbol2$$verify
    0x10000113e <+126>: movq   %rax, -0x28(%rbp)
    0x100001142 <+130>: xorl   %eax, %eax
    0x100001144 <+132>: addq   $0x30, %rsp
    0x100001148 <+136>: popq   %rbp
    0x100001149 <+137>: retq
    0x10000114a <+138>: nopw   (%rax,%rax)
(lldb)
```

The `verify` function is called at `0x1000010fd` if afterwards it jumps to `0x10000110f` then the number is verified correctly, and if not it jumps to `0x10000112b`. We still want it to ask for a number but completely skip the verification. The crude way is to insert NOPs from `0x1000010f6` to `0x10000110f` because it will arrive straight at the part where it prints `"Correct!"`. That means 25 NOPs.

I created the "nop.py" script to insert NOPs:

```shell
% ./nop.py verify 0x10f6 25
Patching "verify" at offset 4342 with 25 NOPs
Read 14416 bytes
Changing values at 4342 to 4367
Writing new data

% ./verify
Serial number: 83
Correct!
```

_Thus 83 is now deemed correct!_

Let's take a look at the actual contents of the binary now:

```shell
(lldb) dis
verify`___lldb_unnamed_symbol1$$verify:
->  0x1000010c0 <+0>:   pushq  %rbp
    0x1000010c1 <+1>:   movq   %rsp, %rbp
    0x1000010c4 <+4>:   subq   $0x30, %rsp
    0x1000010c8 <+8>:   movq   0xf39(%rip), %rdi         ; (void *)0x00007fff78ba32f8: std::__1::cout
    0x1000010cf <+15>:  leaq   0xe86(%rip), %rsi         ; "Serial number: "
    0x1000010d6 <+22>:  movl   $0x0, -0x4(%rbp)
    0x1000010dd <+29>:  callq  0x100001150               ; ___lldb_unnamed_symbol2$$verify
    0x1000010e2 <+34>:  movq   0xf17(%rip), %rdi         ; (void *)0x00007fff78ba3250: std::__1::cin
    0x1000010e9 <+41>:  leaq   -0x8(%rbp), %rsi
    0x1000010ed <+45>:  movq   %rax, -0x10(%rbp)
    0x1000010f1 <+49>:  callq  0x100001d74               ; symbol stub for: std::__1::basic_istream<char, std::__1::char_traits<char> >::operator>>(int&)
    0x1000010f6 <+54>:  nop
    0x1000010f7 <+55>:  nop
    0x1000010f8 <+56>:  nop
    ...
    0x10000110c <+76>:  nop
    0x10000110d <+77>:  nop
    0x10000110e <+78>:  nop
    0x10000110f <+79>:  movq   0xef2(%rip), %rdi         ; (void *)0x00007fff78ba32f8: std::__1::cout
    0x100001116 <+86>:  leaq   0xe4f(%rip), %rsi         ; "Correct!\n"
    0x10000111d <+93>:  callq  0x100001150               ; ___lldb_unnamed_symbol2$$verify
    0x100001122 <+98>:  movq   %rax, -0x20(%rbp)
    0x100001126 <+102>: jmp    0x100001142               ; <+130>
    0x10000112b <+107>: movq   0xed6(%rip), %rdi         ; (void *)0x00007fff78ba32f8: std::__1::cout
    0x100001132 <+114>: leaq   0xe3d(%rip), %rsi         ; "Incorrect!\n"
    0x100001139 <+121>: callq  0x100001150               ; ___lldb_unnamed_symbol2$$verify
    0x10000113e <+126>: movq   %rax, -0x28(%rbp)
    0x100001142 <+130>: xorl   %eax, %eax
    0x100001144 <+132>: addq   $0x30, %rsp
    0x100001148 <+136>: popq   %rbp
    0x100001149 <+137>: retq
    0x10000114a <+138>: nopw   (%rax,%rax)
```

The 25 NOPs are easily spotted.

Recompile and strip the "verify" program again to get rid of the NOPs.

Another approach is to insert a jump instruction that takes us directly to `0x10000110f`. We want to insert the jump at `0x1000010f6` where we have 3 bytes of room to change, so a relative jump would count from the next instruction (after inserting) to `0x10000110f`. Since we are within a one-byte short jump distance of +/-128 we can use a `JMP rel8` with jump instruction opcode `0xeb` (for other jump instructions take a look [here][x86-jumps]). This leaves one byte dangling which we will fill with a NOP. However, this effectively will change the address of the next instruction after our JMP to be `0x1000010f8`. The distance from that to `0x10000110f` is `0x17=23`, which means we will insert the bytes `eb 17 90` at `0x1000010f6`.

I wrote another script, "patch_bytes.py", that will allow us to insert those bytes:

```shell
% ./patch_bytes.py verify 0x10f6 0xeb 0x17 0x90
Patching "verify" at offset 4342 with: [235, 23, 144] (3 bytes)
Read 14416 bytes
Changing values at 4342 to 4345
Writing new data

% ./verify
Serial number: 83
Correct!
```

Let's take a look at the resulting binary:

```shell
(lldb) dis -b
verify`___lldb_unnamed_symbol1$$verify:
->  0x1000010c0 <+0>:   55                    pushq  %rbp
    0x1000010c1 <+1>:   48 89 e5              movq   %rsp, %rbp
    ...
    0x1000010f6 <+54>:  eb 17                 jmp    0x10000110f               ; <+79>
    0x1000010f8 <+56>:  90                    nop
    0x1000010f9 <+57>:  48 89 45 e8           movq   %rax, -0x18(%rbp)
    ...
    0x10000110f <+79>:  48 8b 3d f2 0e 00 00  movq   0xef2(%rip), %rdi         ; (void *)0x00007fff78ba32f8: std::__1::cout
    0x100001116 <+86>:  48 8d 35 4f 0e 00 00  leaq   0xe4f(%rip), %rsi         ; "Correct!\n"
    0x10000111d <+93>:  e8 2e 00 00 00        callq  0x100001150               ; ___lldb_unnamed_symbol2$$verify
    0x100001122 <+98>:  48 89 45 e0           movq   %rax, -0x20(%rbp)
    0x100001126 <+102>: e9 17 00 00 00        jmp    0x100001142               ; <+130>
    0x10000112b <+107>: 48 8b 3d d6 0e 00 00  movq   0xed6(%rip), %rdi         ; (void *)0x00007fff78ba32f8: std::__1::cout
    0x100001132 <+114>: 48 8d 35 3d 0e 00 00  leaq   0xe3d(%rip), %rsi         ; "Incorrect!\n"
    0x100001139 <+121>: e8 12 00 00 00        callq  0x100001150               ; ___lldb_unnamed_symbol2$$verify
    0x10000113e <+126>: 48 89 45 d8           movq   %rax, -0x28(%rbp)
    0x100001142 <+130>: 31 c0                 xorl   %eax, %eax
    0x100001144 <+132>: 48 83 c4 30           addq   $0x30, %rsp
    0x100001148 <+136>: 5d                    popq   %rbp
    0x100001149 <+137>: c3                    retq
    0x10000114a <+138>: 66 0f 1f 44 00 00     nopw   (%rax,%rax)
```

Take a look in particular at the following to view our inserted instructions:

```shell
0x1000010f6 <+54>:  eb 17                 jmp    0x10000110f               ; <+79>
0x1000010f8 <+56>:  90                    nop
```

Continue to read the [third part]({% post_url 2016-09-07-patching-binaries-live-modification %}) about changing values in a program, and live modification of the memory of a running program.

[x86]: https://en.wikipedia.org/wiki/X86
[x86-jumps]: http://x86.renejeschke.de/html/file_module_x86_id_147.html
[mach-o]: https://en.wikipedia.org/wiki/Mach-O
[lldb]: http://lldb.llvm.org
