---
layout: post
title:  "Sigs: Ambiguous Member Functions"
date:   2016-09-04 13:02:42 +0200
tags:   sigs c++ c++14
---

I recently updated my [sigs][sigs] project (simple thread-safe signal/slot C++ library) with support for ambiguous member functions by use of the `sigs::Use<>::overloadOf()`:

```c++
template <typename... Args>
struct Use {
  template <typename Cls, typename Ret>
  static auto overloadOf(Ret (Cls::*MembFunc)(Args...)) {
    return MembFunc;
  }
};
```

_It makes use of variadic templates and auto return type deduction so the compiler must be C++14 compliant._

To illustrate why it was needed consider this: Sometimes there are several overloads for a given function and then it's not enough to just specify `&Class::functionName` because the compiler does not know which overload to choose.

The `Ambiguous` class has two overloads of `foo`:

```c++
class Ambiguous {
public:
  void foo(int i, int j) { std::cout << "Ambiguous::foo(int, int)\n"; }

  void foo(int i, float j) { std::cout << "Ambiguous::foo(int, float)\n"; }
};
```

Without the new construct the following would result in compilation failure:

```c++
sigs::Signal<void(int, int)> s;

Ambiguous amb;
s.connect(&amb, &Ambiguous::foo); // <-- Will fail!
```

The error would look something similar to this:

```
test.cc:147:7: error: no matching member function for call to 'connect'
    s.connect(&amb, &Ambiguous::foo);
    ~~^~~~~~~
./sigs.h:144:16: note: candidate template ignored: couldn't infer template argument 'MembFunc'
    Connection connect(Instance *instance, MembFunc Instance::*mf) {
```

To fix it we must use `sigs::Use<>::overloadOf()`:

```c++
s.connect(&amb, sigs::Use<int, int>::overloadOf(&Ambiguous::foo));
s(42, 48);

/* Prints:
Ambiguous::foo(int, int)
*/
```

Without changing the signal we can also connect the second overload `foo(int, float)`:

```c++
// This one only works because int can be coerced into float!
s.connect(&amb, sigs::Use<int, float>::overloadOf(&Ambiguous::foo));
s(12, 34);

/* Prints:
Ambiguous::foo(int, int)
Ambiguous::foo(int, float)
*/
```

[sigs]: https://github.com/netromdk/sigs
