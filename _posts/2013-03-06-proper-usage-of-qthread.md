---
layout: post
title:  "Proper usage of QThread"
date:   2013-03-06 16:13:10 +0200
tags:   qt concurrency
---

<strong>QThread</strong> should be viewed as the mediator with the OS of the new thread and not as being the actual thread itself!

- Don't subclass QThread only to extend the actual thread implementation which is hardly ever necessary.
- Create a QObject custom class that does what should be done and has a public slot for processing.
- Create a QThread and custom instance, call `custom->moveToThread(thread)`.
- Connect signals from thread to custom instance, started/finished along with `deleteLater()` invocations.
- Finally call `thread->start()`.

AND very important to not allocate on heap in the constructor of the custom class only on the stack. Objects can be allocated on the heap when the public slot is called, and thus the context is the new thread and not the "home thread".
