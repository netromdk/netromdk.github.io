---
layout: post
title:  "Debugging Smart Pointers"
date:   2016-09-02 23:36:02 +0200
tags:   lldb debugging c++ c++14 clang
---

In recent years I have switched to using [LLDB][lldb] (of the [LLVM][llvm] project) as my primary debugger. In this post I'll show how to debug instances of [std::shared_ptr][sptr] and [std::unique_ptr][uptr], and what the differences are.

Create "pointers.cc" with the following contents:

```c++
#include <memory>

class Test {
public:
  Test() : a(1), b(2), c(3) { }

  int a, b, c;
};

int main() {
  auto sptr = std::make_shared<Test>();
  auto uptr = std::make_unique<Test>();
  return 0;
}
```

Compile the program as `pointers` with debugging symbols and let's load it up with LLDB:

```shell
% lldb ./pointers
(lldb) target create "./pointers"
Current executable set to './pointers' (x86_64).
(lldb) b 13
Breakpoint 1: where = pointers`main + 343 at pointers.cc:13, address = 0x0000000100001357
(lldb) r
Process 65102 launched: './pointers' (x86_64)
Process 65102 stopped
* thread #1: tid = 0x18881d, 0x0000000100001357 pointers`main + 343 at pointers.cc:13, queue = 'com.apple.main-thread', stop reason = breakpoint 1.1
    frame #0: 0x0000000100001357 pointers`main + 343 at pointers.cc:13
   10  	int main() {
   11  	  auto sptr = std::make_shared<Test>();
   12  	  auto uptr = std::make_unique<Test>();
-> 13  	  return 0;
   14  	}
```

We want to find out what the member variables of the two pointers are. If we just print the value it won't help much:

```shell
(lldb) p sptr
(std::__1::shared_ptr<Test>) $0 = std::__1::shared_ptr<Test>::element_type @ 0x00000001001037f8 strong=1 weak=1 {
  __ptr_ = 0x00000001001037f8
}
```

We observe that `sptr` has the member `__ptr_` with address `0x00000001001037f8`, which is how we access its members:

```shell
(lldb) p sptr.__ptr_
(element_type *) $1 = 0x00000001001037f8
(lldb) p *sptr.__ptr_
(std::__1::shared_ptr<Test>::element_type) $2 = (a = 1, b = 2, c = 3)
```

The `::element_type` has type `Test` so the following is equivalent:

```shell
(lldb) p *(Test*)sptr.__ptr_
(Test) $3 = (a = 1, b = 2, c = 3)
```

The address can also be used directly:

```shell
(lldb) p *(Test*)0x00000001001037f8
(Test) $4 = (a = 1, b = 2, c = 3)
```

Now, `uptr` is a little different:

```shell
(lldb) p uptr
(std::__1::unique_ptr<Test, std::__1::default_delete<Test> >) $5 = {
  __ptr_ = {
    std::__1::__libcpp_compressed_pair_imp<Test *, std::__1::default_delete<Test>, 2> = {
      __first_ = 0x0000000100101490
    }
  }
}
```

It also has the `__ptr_` but it doesn't point directly to its value like `sptr.__ptr_` does:

```shell
(lldb) p uptr.__ptr_
(std::__1::__compressed_pair<Test *, std::__1::default_delete<Test> >) $6 = {
  std::__1::__libcpp_compressed_pair_imp<Test *, std::__1::default_delete<Test>, 2> = {
    __first_ = 0x0000000100101490
  }
}
(lldb) p *uptr.__ptr_
error: indirection requires pointer operand ('std::__1::__compressed_pair<Test *, std::__1::default_delete<Test> >' invalid)
```

Instead we have to reference the `__first_` member, but can still use the address directly, of course:

```shell
(lldb) p uptr.__ptr_.__first_
(Test *) $7 = 0x0000000100101490
(lldb) p *uptr.__ptr_.__first_
(Test) $8 = (a = 1, b = 2, c = 3)
(lldb) p *(Test*)0x0000000100101490
(Test) $9 = (a = 1, b = 2, c = 3)
```

[lldb]: http://lldb.llvm.org
[llvm]: http://llvm.org
[sptr]: http://en.cppreference.com/w/cpp/memory/shared_ptr
[uptr]: http://en.cppreference.com/w/cpp/memory/unique_ptr
