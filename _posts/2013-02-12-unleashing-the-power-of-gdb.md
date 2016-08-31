---
layout: post
title:  "Unleashing the Power of GDB"
date:   2013-02-12 01:33:30 +0200
tags:   gdb debugging c++ gcc
---

The GNU Debugger<a href="http://sourceware.org/gdb/current/onlinedocs/gdb/index.html#Top" target="_blank"><sup>[1]</sup></a> is my favorite debugging tool and I personally think it's essential for any *nix developer to know how to use it properly if you're working with C/C++, D, Go, Obj-C, Fortran, Pascal, Modula-2 or Ada<a href="http://sourceware.org/gdb/current/onlinedocs/gdb/Supported-Languages.html#Supported-Languages"><sup>[2]</sup></a>.

<em>This is the first in hopefully a series of entries about utilising GDB to its fullest.</em> First a little about how to invoke GDB, secondly some basic usage of some of the most frequently used commands, how to attach to a running program and modify/redirect its `stdout` and `stderr`, then how to install and use watchpoints, and finally how to manually call functions and class methods.

<h4>Invocation</h4>
Here is how to start debugging a program:

```shell
gdb ./program
```

Make sure to compile your programs with debugging symbols to make source code and line numbers available while debugging. If using GCC or Clang use the `-g` argument to do this.

Sometimes arguments have to be passed to the program to debug:

```shell
gdb --args ./program arg1 arg2 arg3..
```

<h4>Basic usage</h4>
The basic commands to use during the debugging phase is `run` (`r`), `step` (`s`), `next` (`n`), `continue` (`c`), `frame` (`f`), `backtrace` (`bt`), `print` (`p`), `info` (`i`), `break`(`b`), `lines` (`l`), `help` (`h`) and `quit` (`q`).

<ul>
	<li><strong>run</strong> will run the program or re-run if already started.</li>
	<li><strong>step</strong> steps until the program reaches a new source code line and can be used with an integer argument to step that many times.</li>
	<li><strong>next</strong> works like `step` except it will treat a function call like a single instruction. Can also be given an integer argument.</li>
	<li><strong>continue</strong> will resume execution after hitting a breakpoint, for instance.</li>
	<li><strong>frame</strong> can be used to select or print current stack frame. When no argument is given it will print the current stack frame, and otherwise it will select the chosen frame.</li>
	<li><strong>backtrace</strong> is used to print a backtrace of all stack frames in order. When an argument is given it will print the innermost frames if positive and the outermost if negative.</li>
	<li><strong>print</strong> prints the value of an expression. It could be variables in the current stack frame, globals or an entire file. As we shall see later on it can also be used to invoke system calls and more.</li>
	<li><strong>info</strong> is very useful because it can display a lot of information about variables, threads, frame, registers, files, types, symbols, locals, addresses and tons more.</li>
	<li><strong>break</strong> is used to set breakpoints in the program at a specified line, function or address.</li>
	<li><strong>lines</strong> with no arguments shows the 10 source code lines centered at the current line. The argument can be a line number, a file with line number (FILE:LINENUM), a function, a file and function (FILE:FUNCTION) or an address (*ADDR).</li>
	<li><strong>help</strong> is invaluable because it explains about more or less everything in GDB. <em>So use it!</em></li>
	<li>Lastly <strong>quit</strong> exits the program and terminates GDB.</li>
</ul>

So let's give them a spin. Save the following code to a file named "test.cpp":

```c++
#include <string>
#include <iostream>
using namespace std;

void func2(const string &str) {
  cout << "func2 says: " << str << endl;
}

void func1(int i) {
  string str(i, "$");
  func2(str);
}

int main(int argc, char **argv) {
  for (int i = 1; i <= 3; i++) {
    func1(i * 5);
  }
  return 0;
}
```

Then compile it thusly:

```shell
g++ -g -o test test.cpp
```

And debug it:

```shell
gdb ./test
```

First thing we will do is set a breakpoint at `main` and run the program:

```shell
(gdb) b main
Breakpoint 1 at 0x100000c4f: file test.cpp, line 15.
(gdb) r
Starting program: /private/tmp/test 
Reading symbols for shared libraries ++............................. done

Breakpoint 1, main (argc=1, argv=0x7fff5fbffa80) at test.cpp:15
15	  for (int i = 1; i <= 3; i++) {
(gdb)
```

Then we inspect the arguments of the current stack:

```shell
(gdb) f
#0  main (argc=1, argv=0x7fff5fbffa80) at test.cpp:15
15	  for (int i = 1; i <= 3; i++) {
(gdb) i args
argc = 1
argv = (char **) 0x7fff5fbffa80
(gdb) p argv
$1 = (char **) 0x7fff5fbffa80
(gdb) p *argv
$2 = 0x7fff5fbffbf8 "/private/tmp/test"
(gdb) p argv[0]
$3 = 0x7fff5fbffbf8 "/private/tmp/test"
(gdb) p argv[1]
$4 = 0x0
(gdb)
```

Notice that the manipulation of the `argv` array is very much like in C/C++ with deference/element access (`p *argv` and `p argv[1]`).

Now we step one round through the `for`-loop:

```shell
(gdb) s
16	    func1(i * 5);
(gdb) s
func1 (i=5) at test.cpp:10
10	  string str(i, "$");
(gdb) s
11	  func2(str);
(gdb) s
func2 (str=@0x7fff5fbffa00) at test.cpp:6
6	  cout << "func2 says: " << str << endl;
(gdb) s
func2 says: $$$$$
7	}
(gdb) s
func1 (i=5) at test.cpp:12
12	}
(gdb) s
0x0000000100000bee	11	  func2(str);
(gdb) s
main (argc=1, argv=0x7fff5fbffa80) at test.cpp:15
15	  for (int i = 1; i <= 3; i++) {
(gdb)
```

What happens is that we first call `func1(5)` which in turn calls `func2(str)` and returns to the `for`-loop. If we instead used `next` the output would be shorter because it would not step into `func1`:

```shell
(gdb) n
16	    func1(i * 5);
(gdb) n
func2 says: $$$$$$$$$$
15	  for (int i = 1; i <= 3; i++) {
(gdb)
```

It's time to see a backtrace in action:

```shell
(gdb) b func2
Breakpoint 2 at 0x100000abc: file test.cpp, line 6.
(gdb) c
Continuing.

Breakpoint 2, func2 (str=@0x7fff5fbffa00) at test.cpp:6
6	  cout << "func2 says: " << str << endl;
(gdb) bt
#0  func2 (str=@0x7fff5fbffa00) at test.cpp:6
#1  0x0000000100000b81 in func1 (i=15) at test.cpp:11
#2  0x0000000100000c65 in main (argc=1, argv=0x7fff5fbffa80) at test.cpp:16
(gdb)
```

The trace shows the stack frames in a numbered fashion from `func2` and backwards. Let's inspect frame `#1`:

```shell
(gdb) f 1
#1  0x0000000100000b81 in func1 (i=15) at test.cpp:11
11	  func2(str);
(gdb) l
6	  cout << "func2 says: " << str << endl;
7	}
8	
9	void func1(int i) {
10	  string str(i, "$");
11	  func2(str);
12	}
13	
14	int main(int argc, char **argv) {
15	  for (int i = 1; i <= 3; i++) {
(gdb)
```

That concludes the basics:

```shell
(gdb) q
The program is running.  Exit anyway? (y or n) y
```

<h4>Attaching live programs</h4> 
GDB can attach to running programs by stating its process ID (PID):

```shell
gdb -p PID
```

Here is a little trick to debug a program started by another program at runtime. It might be crucial to attach and debug "from the top" so what to do? Simply add a little sleep to leave enough time to find the PID and attach using GDB. After attaching set the needed breakpoints and `continue` execution. Another way of achieving the same thing is to use:

```shell
gdb --waitfor=PROCNAME
```

Where PROCNAME is the process name to continuously poll for until it has been launched. Note that some instructions <em>will</em> have been executed before GDB attaches this way because polling is not exactly instant however close it might seem to be.

A very useful technique is to know how to redirect `stdout` (file descriptor 1) and/or `stderr` (FD 2) after a program has started. We are going to exploit the fact that `print` can invoke system calls like `dup2` and `open` in our case. After attaching to the process do the following:

```shell
(gdb) p (void) dup2((int) open("/tmp/out.txt", 0x201, 0640), 1)
(gdb) p (void) dup2((int) open("/tmp/err.txt", 0x201, 0640), 2)
(gdb) detach
(gdb) q
```

In short it will redirect `stdout` to "/tmp/stdout.txt" and `stderr` to "/tmp/stderr.txt". The
system call `open` is used to open a file for writing in our case. The mode "0x201" actually means
"write only and create file if nonexistent" since `O_CREAT | O_WRONLY` = 0x200 | 0x1 = 0x201 (see
the `fcntl.h` header file for details). "0640" is the umask (user has RW and group has R). After
opening a file and retrieving its FD we need to redirect the device in question to it (`stdout` and
`stderr` in this case). This is achieved using `dup2` that creates an alias to the FD, does
redirection and closes the old FD.

Additionally, it's important to cast types to enforce GDB to behave correctly. If this is not the case it could argue giving the following error message: 

<p><blockquote>Unable to call function "open" at 0x7fff906e3fe4: no return type information available.
To call this function anyway, you can cast the return type explicitly (e.g. ''print (float) fabs (3.0)'')</blockquote></p>

Another scenario would be to completely turn off output to `stdout`/`stderr`:

```shell
(gdb) p (void) close(1)
(gdb) p (void) close(2)
(gdb) det
(gdb) q
```

Here is some code for the testing the above instructions and to observe that the output is redirected or stopped:

```c++
#include <unistd.h>
#include <iostream>
using namespace std;

int main(int argc, char **argv) {
  for (int i = 0; i < 120; i++) {
    cout << "hello stdout" << endl;
    cerr << "hello stderr" << endl;
    sleep(2);
  }
  return 0;
}
```

<h4>Watchpoints</h4>
Watchpoints are useful when certain variables need to be watched using the command `watch`. Whenever a watched variable is changed GDB will show the old and new value along with the stack frame. Try debugging the program from before:

```shell
(gdb) b main
Breakpoint 1 at 0x100000c4f: file test.cpp, line 15.
(gdb) r  
Starting program: /private/tmp/test 
Reading symbols for shared libraries ++............................. done

Breakpoint 1, main (argc=1, argv=0x7fff5fbffa80) at test.cpp:15
15	  for (int i = 1; i <= 3; i++) {
(gdb) watch i
Hardware watchpoint 2: i
(gdb) d 1
(gdb) c
Continuing.
Hardware watchpoint 2: i

Old value = 0
New value = 1
0x0000000100000c56 in main (argc=1, argv=0x7fff5fbffa80) at test.cpp:15
15	  for (int i = 1; i <= 3; i++) {
(gdb) c
Continuing.
$$$$$
Hardware watchpoint 2: i

Old value = 1
New value = 2
0x0000000100000c6e in main (argc=1, argv=0x7fff5fbffa80) at test.cpp:15
15	  for (int i = 1; i <= 3; i++) {
(gdb) c
Continuing.
$$$$$$$$$$
Hardware watchpoint 2: i

Old value = 2
New value = 3
0x0000000100000c6e in main (argc=1, argv=0x7fff5fbffa80) at test.cpp:15
15	  for (int i = 1; i <= 3; i++) {
(gdb) c
Continuing.
$$$$$$$$$$$$$$$
Hardware watchpoint 2: i

Old value = 3
New value = 4
0x0000000100000c6e in main (argc=1, argv=0x7fff5fbffa80) at test.cpp:15
15	  for (int i = 1; i <= 3; i++) {
(gdb) c
Continuing.

Watchpoint 2 deleted because the program has left the block in
which its expression is valid.
```

Notice I did `d 1` which means to delete the breakpoint with number "1". 

<h4>Calling functions</h4>
While debugging it is possible to call functions of the program - even pass them arguments - and have the result saved in value history if non-void. Load the first test program from above in GDB:

```shell
(gdb) b 11
Breakpoint 1 at 0x100000b45: file test.cpp, line 11.
(gdb) r
Starting program: /private/tmp/test 
Reading symbols for shared libraries ++............................. done

Breakpoint 1, func1 (i=5) at test.cpp:11
11	  func2(str);
(gdb) i loc   
str = {
  _M_dataplus = {
    <std::allocator<char>> = {
      <__gnu_cxx::new_allocator<char>> = {<No data fields>}, <No data fields>}, 
    members of std::string::_Alloc_hider: 
    _M_p = 0x100103938 "$$$$$"
  }
}
(gdb) call func2(str)
func2 says: $$$$$
(gdb) q
The program is running.  Exit anyway? (y or n) y
```

Since we already have a `std::string` object we can call `func2(str)` manually.

This functionality opens up for a variety of different uses. One is to use predefined debugging functions to display custom data structures that GDB doesn't know about. Take the `std::string` structure above for `str` - it's not very descriptive except for the fact that its content is visible (`_M_p = 0x100103938 "$$$$$"`). 

Compile and load up the following program in GDB:

```c++
#include <string>
#include <iostream>
using namespace std;

class Test {
public:
  Test() : a(1), b(2), c(3), str("Hello, World!") { }

  int a, b, c;
  string str;
};

void dbg(const Test &test) {
  cout << "a = " << test.a << endl
       << "b = " << test.b << endl
       << "c = " << test.c << endl
       << "str = " << test.str << endl;
}

int main(int argc, char **argv) {
  Test test;
  return 0;
}
```

Then it's high time to compare the two approaches:

```shell
(gdb) b main
Breakpoint 1 at 0x100000b31: file custom.cpp, line 21.
(gdb) r
Starting program: /private/tmp/custom 
Reading symbols for shared libraries ++............................. done

Breakpoint 1, main (argc=1, argv=0x7fff5fbffa78) at custom.cpp:21
21	  Test test;
(gdb) n
22	  return 0;
(gdb) call dbg(test)
a = 1
b = 2
c = 3
str = Hello, World!
(gdb) p test
$1 = {
  a = 1, 
  b = 2, 
  c = 3, 
  str = {
    _M_dataplus = {
      <std::allocator<char>> = {
        <__gnu_cxx::new_allocator<char>> = {<No data fields>}, <No data fields>}, 
      members of std::string::_Alloc_hider: 
      _M_p = 0x100103938 "Hello, World!"
    }
  }
}
(gdb)
```

Keep in mind that this is just a simple test. The great thing about this is that you can fine-tune the debugging function to show only what you need to see and leave out the unnecessary parts. <em>And make sure the variables you're accessing are actually live and initialised when doing so!</em> If we skip the `next` command above this is the result:

```shell
(gdb) call dbg(test)
a = 0
b = 0
c = 0

Program received signal EXC_BAD_ACCESS, Could not access memory.
Reason: KERN_INVALID_ADDRESS at address: 0xffffffffffffffe8
0x00007fff8fa68c85 in std::operator<< <char, std::char_traits<char>, std::allocator<char> > ()
```

It is also possible to call class methods. Add the following snippet after the constructor on line 7 in the previous code:

```c++
void __attribute__ ((used)) dump() {
  cout << "str = " << str << endl;
}
```

Notice the use of the "used" attribute - this is because if the method is not called in the source code, which is the case here, the compiler will optimise the whole thing away in most cases.

Invoking the method on our `test` object is easy:

```shell
(gdb) call test.dump()
str = Hello, World!
```

That concludes the basic knowledge needed to start debugging using GDB. Stay tuned for more!
