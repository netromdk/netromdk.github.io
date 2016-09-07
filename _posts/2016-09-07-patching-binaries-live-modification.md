---
layout: post
title:  "Patching Binaries: Live Modification"
date:   2016-09-07 23:35:59 +0200
tags:   patching x86 c++ python lldb disassembling
---

_Patching Binaries_ is a series of articles about how to extract information and modify program behavior. It focuses on the Mac [Mach-O][mach-o] executable format for the [x86][x86] architecture, but the techniques are similar for other formats.

Before proceeding it is recommended reading the [second part]({% post_url 2016-09-06-patching-binaries-jumping-the-fence %}) about jumping and bypassing instructions.

_The files used in this article can be found [here](https://github.com/netromdk/patching/tree/master/live)._

This article is about how to change data values of a program and how to read from and write to memory of a live program to change its behavior.

Compile "score.cc" as "score" and strip it. Let's see what it does:

```shell
% ./score
score = 1
```

Think of a game that displays your high score. We want to change it so use "lldb_entrypoint.sh" to start it up:

```shell
% ./lldb_entrypoint.sh score
(lldb) target create "score"
Current executable set to 'score' (x86_64).
(lldb) command source -s 0 '/tmp/.lldbcmds'
Executing commands in '/tmp/.lldbcmds'.
(lldb) b 0x0000000100001140
Breakpoint 1: address = 0x0000000100001140
(lldb) r
Process 38693 launched: '/Users/netrom/git/patching/live/score' (x86_64)
Process 38693 stopped
* thread #1: tid = 0x28e293, 0x0000000100001140 score`___lldb_unnamed_function1$$score, queue = 'com.apple.main-thread', stop reason = breakpoint 1.1
    frame #0: 0x0000000100001140 score`___lldb_unnamed_function1$$score
score`___lldb_unnamed_function1$$score:
->  0x100001140 <+0>: pushq  %rbp
    0x100001141 <+1>: movq   %rsp, %rbp
    0x100001144 <+4>: subq   $0x10, %rsp
    0x100001148 <+8>: movq   0xeb1(%rip), %rdi         ; (void *)0x00007fff78ba32f8: std::__1::cout
(lldb) dis -b
score`___lldb_unnamed_function1$$score:
->  0x100001140 <+0>:  55                    pushq  %rbp
    0x100001141 <+1>:  48 89 e5              movq   %rsp, %rbp
    0x100001144 <+4>:  48 83 ec 10           subq   $0x10, %rsp
    0x100001148 <+8>:  48 8b 3d b1 0e 00 00  movq   0xeb1(%rip), %rdi         ; (void *)0x00007fff78ba32f8: std::__1::cout
    0x10000114f <+15>: 48 8d 35 26 0e 00 00  leaq   0xe26(%rip), %rsi         ; "score = "
    0x100001156 <+22>: c7 45 fc 00 00 00 00  movl   $0x0, -0x4(%rbp)
    0x10000115d <+29>: c7 45 f8 01 00 00 00  movl   $0x1, -0x8(%rbp)
    0x100001164 <+36>: e8 37 00 00 00        callq  0x1000011a0               ; ___lldb_unnamed_function2$$score
    0x100001169 <+41>: 8b 75 f8              movl   -0x8(%rbp), %esi
    0x10000116c <+44>: 48 89 c7              movq   %rax, %rdi
    0x10000116f <+47>: e8 2c 0c 00 00        callq  0x100001da0               ; symbol stub for: std::__1::basic_ostream<char, std::__1::char_traits<char> >::operator<<(int)
    0x100001174 <+52>: 48 8d 35 0a 0e 00 00  leaq   0xe0a(%rip), %rsi         ; "'\n'"
    0x10000117b <+59>: 48 89 c7              movq   %rax, %rdi
    0x10000117e <+62>: e8 1d 00 00 00        callq  0x1000011a0               ; ___lldb_unnamed_function2$$score
    0x100001183 <+67>: 31 c9                 xorl   %ecx, %ecx
    0x100001185 <+69>: 48 89 45 f0           movq   %rax, -0x10(%rbp)
    0x100001189 <+73>: 89 c8                 movl   %ecx, %eax
    0x10000118b <+75>: 48 83 c4 10           addq   $0x10, %rsp
    0x10000118f <+79>: 5d                    popq   %rbp
    0x100001190 <+80>: c3                    retq
```

The score of value `1` is passed to to `-0x8(%rbp)` at `0x10000115d`. At offset `0x115d+3=0x1160` an 32-bit integer is passed as 4 bytes `1 0 0 0` (little endian).

Let's say we want to change this value to `0x4030201 = 67305985`. First the number must be split up into the 4 bytes: `1 2 3 4` (remember the reverse order here!). Then we can patch the program:

```shell
% ./patch_bytes.py score 0x1160 1 2 3 4
Patching "score" at offset 4448 with: [1, 2, 3, 4] (4 bytes)
Read 14348 bytes
Changing values at 4448 to 4452
Writing new data

% ./score
score = 67305985
```

Inspecting the instructions yields what we expect:

```shell
(lldb) dis -b -c 1 -s 0x10000115d
score`___lldb_unnamed_function1$$score:
    0x10000115d <+29>: c7 45 f8 01 02 03 04  movl   $0x4030201, -0x8(%rbp)    ; imm = 0x4030201
```

Recompile and strip "score.cc" to get back to square one.

**We can also read and write registers of a program while it is running!**

So let's try changing the score right after it is set to `1` but before being used. As we saw before the value is saved into `-0x8(%rbp)` which we first have to understand what it means. The _RBP_ (base pointer) register holds the pointer to the beginning of a stack frame while executing a function. Allocating on the stack is done by using a proper offset from the RBP, and since addresses grow upwards we use a negative addressing of `-0x8` in this case.

In order to go further we need the value of the RBP register:

```shell
% lldb score
(lldb) target create "score"
Current executable set to 'score' (x86_64).
(lldb) b 0x100001164
Breakpoint 1: address = 0x0000000100001164
(lldb) r
Process 40054 launched: '/Users/netrom/git/patching/live/score' (x86_64)
Process 40054 stopped
* thread #1: tid = 0x298ab1, 0x0000000100001164 score`___lldb_unnamed_function1$$score + 36, queue = 'com.apple.main-thread', stop reason = breakpoint 1.1
    frame #0: 0x0000000100001164 score`___lldb_unnamed_function1$$score + 36
score`___lldb_unnamed_function1$$score:
->  0x100001164 <+36>: callq  0x1000011a0               ; ___lldb_unnamed_function2$$score
    0x100001169 <+41>: movl   -0x8(%rbp), %esi
    0x10000116c <+44>: movq   %rax, %rdi
    0x10000116f <+47>: callq  0x100001da0               ; symbol stub for: std::__1::basic_ostream<char, std::__1::char_traits<char> >::operator<<(int)
(lldb) reg read rbp
     rbp = 0x00007fff5fbff5e0
```

Then we want to verify that `-0x8(%rbp)` has value `1`:

```shell
(lldb) p *(int*)(0x00007fff5fbff5e0 - 0x8)
(int) $0 = 1
```

_That might look obscure but the command means to print the deferenced value of an integer pointer address at `-0x8(%rbp)`._

Changing the value can be done in the following way:

```shell
(lldb) p *(int*)(0x00007fff5fbff5e0 - 0x8) = 42
(int) $1 = 42
```

Letting the program continue will show that it worked:

```shell
(lldb) c
Process 40054 resuming
score = 42
Process 40054 exited with status = 0 (0x00000000)
```

_Note that the way to change the value could also have been written as:_

```
(lldb) p *(int*)(`$rbp`-0x8) = 42
```

Which makes it possible to read the register without copying the address by hand!

How much easier would it be if we could automate this? A lot, I agree. So create a file called "setvalue.lldb" with the following contents:

```
b 0x100001164
r
p *(int*)(`$rbp`-0x8) = 42
c
exit
```

And run it like this:

```shell
% lldb -s setvalue.lldb score
(lldb) target create "score"
Current executable set to 'score' (x86_64).
(lldb) command source -s 0 'setvalue.lldb'
Executing commands in '/Users/netrom/git/patching/live/setvalue.lldb'.
(lldb) b 0x100001164
Breakpoint 1: address = 0x0000000100001164
(lldb) r
Process 40286 stopped
* thread #1: tid = 0x29c96b, 0x0000000100001164 score`___lldb_unnamed_function1$$score + 36, queue = 'com.apple.main-thread', stop reason = breakpoint 1.1
    frame #0: 0x0000000100001164 score`___lldb_unnamed_function1$$score + 36
score`___lldb_unnamed_function1$$score:
->  0x100001164 <+36>: callq  0x1000011a0               ; ___lldb_unnamed_function2$$score
    0x100001169 <+41>: movl   -0x8(%rbp), %esi
    0x10000116c <+44>: movq   %rax, %rdi
    0x10000116f <+47>: callq  0x100001da0               ; symbol stub for: std::__1::basic_ostream<char, std::__1::char_traits<char> >::operator<<(int)

Process 40286 launched: '/Users/netrom/git/patching/live/score' (x86_64)
(lldb) p *(int*)(`$rbp`-0x8) = 42
(int) $1 = 42
(lldb) c
score = 42
Process 40286 resuming
Process 40286 exited with status = 0 (0x00000000)

(lldb) exit
```

Another approach is to start LLDB in a wait-for-process state where it will attach once a program with the given process name has been started. Something to note, though, is that the address `0x100001164` of before won't work in this example due to how it attaches to the process. Instead we use the fact that `main()` is the first function so we can name it in LLDB as `___lldb_unnamed_function1$$score`. Also, since it attaches to a running program we don't have to start the process so we just continue after setting the break point. Finally, because we know that the original `0x100001164` is 36 instructions after the start of `main()` we set a break point using the RIP (instruction pointer) register with `$rip+36`.

Create file "waitsetvalue.lldb" with the following contents:

```
b score`___lldb_unnamed_function1$$score
c
b `$rip+36`
c
p *(int*)(`$rbp`-0x8) = 42
c
exit
```

Next we fire up LLDB in one terminal (note it has to run as root/sudo to have privileges):

```shell
% sudo lldb -w -s waitsetvalue.lldb -n score
(lldb) process attach --name "score" --waitfor
```

It is now in a waiting mode, so in the second terminal we simply start the program:

```shell
% ./score
```

LLDB will immediately attach in the other terminal:

```shell
(lldb) command source -s 0 'waitsetvalue.lldb'
Executing commands in '/Users/netrom/git/patching/live/waitsetvalue.lldb'.
(lldb) b score`___lldb_unnamed_function1$$score
Breakpoint 1: where = score`___lldb_unnamed_function1$$score, address = 0x000000010ec6c140
(lldb) c
Process 44506 resuming
Process 44506 stopped
* thread #1: tid = 0x2a93a3, 0x000000010ec6c140 score`___lldb_unnamed_function1$$score, queue = 'com.apple.main-thread', stop reason = breakpoint 1.1
    frame #0: 0x000000010ec6c140 score`___lldb_unnamed_function1$$score
score`___lldb_unnamed_function1$$score:
->  0x10ec6c140 <+0>: pushq  %rbp
    0x10ec6c141 <+1>: movq   %rsp, %rbp
    0x10ec6c144 <+4>: subq   $0x10, %rsp
    0x10ec6c148 <+8>: movq   0xeb1(%rip), %rdi         ; (void *)0x00007fff78ba32f8: std::__1::cout

(lldb) b `$rip+36`
Breakpoint 2: where = score`___lldb_unnamed_function1$$score + 36, address = 0x000000010ec6c164
(lldb) c
Process 44506 resuming
Process 44506 stopped
* thread #1: tid = 0x2a93a3, 0x000000010ec6c164 score`___lldb_unnamed_function1$$score + 36, queue = 'com.apple.main-thread', stop reason = breakpoint 2.1
    frame #0: 0x000000010ec6c164 score`___lldb_unnamed_function1$$score + 36
score`___lldb_unnamed_function1$$score:
->  0x10ec6c164 <+36>: callq  0x10ec6c1a0               ; ___lldb_unnamed_function2$$score
    0x10ec6c169 <+41>: movl   -0x8(%rbp), %esi
    0x10ec6c16c <+44>: movq   %rax, %rdi
    0x10ec6c16f <+47>: callq  0x10ec6cda0               ; symbol stub for: std::__1::basic_ostream<char, std::__1::char_traits<char> >::operator<<(int)

(lldb) p *(int*)(`$rbp`-0x8) = 42
(int) $2 = 42
(lldb) c
Process 44506 resuming
Process 44506 exited with status = 0 (0x00000000)

(lldb) exit
```

Finally, in the second terminal, it will output:

```
score = 42
```

Phew! That concludes data manipulation of live processes.

[x86]: https://en.wikipedia.org/wiki/X86
[mach-o]: https://en.wikipedia.org/wiki/Mach-O
[lldb]: http://lldb.llvm.org
