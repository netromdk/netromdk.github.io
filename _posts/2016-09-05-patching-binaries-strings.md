---
layout: post
title:  "Patching Binaries: Strings"
date:   2016-09-05 22:31:40 +0200
tags:   patching x86 c++ python lldb disassembling
---

_Patching Binaries_ will be a series of articles about how to extract information and modify program behavior. It focuses on the Mac [Mach-O][mach-o] executable format for the [x86][x86] architecture, but the techniques are similar for other formats.

_The files used in this article can be found [here](https://github.com/netromdk/patching/tree/master/strings)._

One of the first things to look at in a binary, when trying to determine information about it, is the constant strings saved within it. In Mach-O binaries these can be found in section `__cstring` of segment `__TEXT`.

Compile the example program "test.cc" as "test" and run it:

```shell
% ./test
std::string example.
And a char* string, too.
```

To list a programs' strings there are at least two programs to help us do that; `otool` and
`strings`.

The most simple is using `strings`:

```shell
% strings -o -t x test
1f50 std::string example.
1f65 And a char* string, too.
```

It shows the strings and their hexadecimal offsets in the binary.

A little more detail can be obtained with `otool` though:

```shell
% otool -s __TEXT __cstring test
test:
Contents of (__TEXT,__cstring) section
0000000100001f50   73 74 64 3a 3a 73 74 72 69 6e 67 20 65 78 61 6d
0000000100001f60   70 6c 65 2e 00 41 6e 64 20 61 20 63 68 61 72 2a
0000000100001f70   20 73 74 72 69 6e 67 2c 20 74 6f 6f 2e 00 0a 00

% otool -v -s __TEXT __cstring test
test:
Contents of (__TEXT,__cstring) section
0000000100001f50   std::string example.
0000000100001f65   And a char* string, too.
0000000100001f7e   \n
```

To understand the layout of the binary block above the strings are end-to-end:

```
0000000100001f50   73 74 64 3a 3a 73 74 72 69 6e 67 20 65 78 61 6d
0000000100001f60   70 6c 65 2e 00

0000000100001f65   41 6e 64 20 61 20 63 68 61 72 2a 20 73 74 72 69
0000000100001f75   6e 67 2c 20 74 6f 6f 2e 00
                    
0000000100001f7e   0a 00
```

_Notice the zero byte for termination at the end of each string!_

We now know that there are two "interesting" strings at offset `1f50` and `1f65`, which means if we read the binary's data at those positions we will find the strings.

The next natural thing to do is modifying the strings. I wrote a python script "patch.py" to patch a binary at an offset with a new string.

Let's change the first string to become `"Hello, World!"`:

```shell
% ./patch.py test 0x1f50 "Hello, World!       "
Patching "test" at offset 8016 with: "Hello, World!       " (20 bytes)
Read 15392 bytes
Size of string at 8016 is 20 bytes
Changing values at 8016 to 8036
Writing new data

% ./test
Hello, World!
And a char* string, too.
```

Voila!

Note that the size of the string we patched is 20 bytes, which means the value to overwrite it cannot be longer than that. To visually clear the old value we put spaces as padding at the end. 

The contents of the binary are now changed to the following:

```shell
% otool -s __TEXT __cstring test
test:
Contents of (__TEXT,__cstring) section
0000000100001f50   48 65 6c 6c 6f 2c 20 57 6f 72 6c 64 21 20 20 20
0000000100001f60   20 20 20 20 00 41 6e 64 20 61 20 63 68 61 72 2a
0000000100001f70   20 73 74 72 69 6e 67 2c 20 74 6f 6f 2e 00 0a 00
```

Notice the 7 spaces (byte `20`) after our string.

As a second example compile "password.cc" into "password" and run it:

```shell
% ./password
Password: test
Invalid!
Password: password
Invalid!
Password: 1234
Invalid!
Password: ^C
```

Okay, so maybe we can't just outright guess it. Let's look at the strings in the binary instead for clues:

```shell
% otool -v -s __TEXT __cstring password
password:
Contents of (__TEXT,__cstring) section
0000000100001f28  Password:
0000000100001f33  AK9FJ31P
0000000100001f3c  You've entered the correct password!\n
0000000100001f62  Invalid!\n
```

One value looks curious: "AK9FJ31P"

```shell
% ./password
Password: AK9FJ31P
You've entered the correct password!
```

_Keep in mind that this is a very simple example and nothing akin to a real-world example._ It just illustrates the technique of finding constant strings in a binary that can be used in different ways.

Lastly we will briefly see how the offsets to the constant strings relate to the machine code instructions of the executable. We will do this by using the debugger [LLDB][lldb] to find the spot where the password of the previous example is loaded into memory.

We start by running LLDB and braking when the `main()` is called:

```shell
% lldb ./password
(lldb) target create "./password"
Current executable set to './password' (x86_64).
(lldb) b main
Breakpoint 1: where = password`main, address = 0x0000000100000650
(lldb) r
Process 84336 launched: './password' (x86_64)
Process 84336 stopped
* thread #1: tid = 0x212d03, 0x0000000100000650 password`main, queue = 'com.apple.main-thread', stop reason = breakpoint 1.1
    frame #0: 0x0000000100000650 password`main
password`main:
->  0x100000650 <+0>:  pushq  %rbp
    0x100000651 <+1>:  movq   %rsp, %rbp
    0x100000654 <+4>:  subq   $0xd0, %rsp
    0x10000065b <+11>: xorl   %esi, %esi
```

Since we do not know the address of the instructions that loads the string we have to step through the program until we find it. To get a better overview we will disassemble the `main()`:

```shell
(lldb) dis
password`main:
->  0x100000650 <+0>:   pushq  %rbp
    0x100000651 <+1>:   movq   %rsp, %rbp
    0x100000654 <+4>:   subq   $0xd0, %rsp
    0x10000065b <+11>:  xorl   %esi, %esi
    0x10000065d <+13>:  movl   $0x18, %eax
......
    0x1000006fb <+171>: jmp    0x1000006d7               ; <+135>
    0x100000700 <+176>: jmp    0x100000705               ; <+181>
    0x100000705 <+181>: movq   0x18fc(%rip), %rdi        ; (void *)0x00007fff78ba32f8: std::__1::cout
    0x10000070c <+188>: leaq   0x1815(%rip), %rsi        ; "Password: "
    0x100000713 <+195>: callq  0x100000820               ; std::__1::basic_ostream<char, std::__1::char_traits<char> >& std::__1::operator<<<std::__1::char_traits<char> >(std::__1::basic_ostream<char, std::__1::char_traits<char> >&, char const*)
    0x100000718 <+200>: movq   %rax, -0xa8(%rbp)
    0x10000071f <+207>: jmp    0x100000724               ; <+212>
    0x100000724 <+212>: movq   0x18d5(%rip), %rdi        ; (void *)0x00007fff78ba3250: std::__1::cin
......
    0x100000737 <+231>: movq   %rax, -0xb0(%rbp)
    0x10000073e <+238>: jmp    0x100000743               ; <+243>
    0x100000743 <+243>: leaq   0x17e9(%rip), %rax        ; "AK9FJ31P"
......
```

From the above excerpt it is shown that the `"Password: "` text is loaded at `0x10000070c`, and that the input of `std::cin` at `0x100000724` is compared with the real password `"AK9FJ31P"` on `0x100000743`. But let's look closer at that last address:

```shell
(lldb) dis -b -c 1 -s 0x100000743
password`main:
    0x100000743 <+243>: 48 8d 05 e9 17 00 00  leaq   0x17e9(%rip), %rax        ; "AK9FJ31P"
```

We know that `"AK9FJ31P"` resides at offset `1f33` but here it uses `17e9`, why? To understand this it is necessary to know that it uses relative addressing. So the `leaq` (load effective address) is invoked at `743` and if we add that to `17e9` then we get `1f2c`. That's still not the correct offset! However, looking at the assembly code we see that it uses 7 bytes, and now the equation fits: `743+7+17e9=1f33`!

_Note that we used an unstripped binary for this example, which means it still contains useful information. However, with a real-world example it will most likely be stripped of symbols._

Let's remove the symbols:

```shell
% strip password
```

Now when we try to hook up on the `main()` it is evident that we can't because the symbol is not known:

```shell
% lldb ./password
(lldb) target create "./password"
Current executable set to './password' (x86_64).
(lldb) b main
Breakpoint 1: no locations (pending).
WARNING:  Unable to resolve breakpoint to any actual locations.
```

Instead we will run the program until it asks for the password and then break to see a backtrace to get our bearings:

```shell
(lldb) r
Process 86636 launched: './password' (x86_64)
Password: Process 86636 stopped
* thread #1: tid = 0x225927, 0x00007fff9218df8a libsystem_kernel.dylib`__read_nocancel + 10, stop reason = signal SIGSTOP
    frame #0: 0x00007fff9218df8a libsystem_kernel.dylib`__read_nocancel + 10
libsystem_kernel.dylib`__read_nocancel:
->  0x7fff9218df8a <+10>: jae    0x7fff9218df94            ; <+20>
    0x7fff9218df8c <+12>: movq   %rax, %rdi
    0x7fff9218df8f <+15>: jmp    0x7fff921887cd            ; cerror_nocancel
    0x7fff9218df94 <+20>: retq
(lldb) bt
* thread #1: tid = 0x225927, 0x00007fff9218df8a libsystem_kernel.dylib`__read_nocancel + 10, stop reason = signal SIGSTOP
  * frame #0: 0x00007fff9218df8a libsystem_kernel.dylib`__read_nocancel + 10
    frame #1: 0x00007fff8d103155 libsystem_c.dylib`_sread + 16
    frame #2: 0x00007fff8d102769 libsystem_c.dylib`__srefill1 + 24
    frame #3: 0x00007fff8d102884 libsystem_c.dylib`__srget + 14
    frame #4: 0x00007fff8d0fe52b libsystem_c.dylib`getc + 52
    frame #5: 0x00007fff8dd38051 libc++.1.dylib`std::__1::__stdinbuf<char>::__getchar(bool) + 119
    frame #6: 0x00007fff8dd2ee4c libc++.1.dylib`std::__1::basic_istream<char, std::__1::char_traits<char> >::sentry::sentry(std::__1::basic_istream<char, std::__1::char_traits<char> >&, bool) + 200
    frame #7: 0x000000010000089e password`___lldb_unnamed_symbol3$$password + 46
    frame #8: 0x0000000100000737 password`___lldb_unnamed_symbol1$$password + 231
    frame #9: 0x00007fff9a14a5ad libdyld.dylib`start + 1
```

The backtrace shows two function calls in our `password` executable. Taking a look at frame 8 shows the same location we used previously to argument about the address of the constant string being loaded:

```frame
(lldb) f 8
frame #8: 0x0000000100000737 password`___lldb_unnamed_symbol1$$password + 231
password`___lldb_unnamed_symbol1$$password:
    0x100000737 <+231>: movq   %rax, -0xb0(%rbp)
    0x10000073e <+238>: jmp    0x100000743               ; <+243>
    0x100000743 <+243>: leaq   0x17e9(%rip), %rax        ; "AK9FJ31P"
    0x10000074a <+250>: leaq   -0x88(%rbp), %rcx
```

[x86]: https://en.wikipedia.org/wiki/X86
[mach-o]: https://en.wikipedia.org/wiki/Mach-O
[lldb]: http://lldb.llvm.org
