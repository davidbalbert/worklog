---
title: "Sketching out a new programming language"
date: 2025-03-30T16:21:47-04:00
draft: true
---

I'm working on a new programming language. It's called [Blip](https://github.com/davidbalbert/blip).[^1]

Right now, it's a collection of losely connected ideas that don't fit together. There's certainly no code you can run. But I think there's something interesting here, and I want to get some feedback.

At some point I'll write a larger overview. But for today, we'll stick to the most interesting ideas I've come up with so far: safe mutable aliasing, and resource management without RAII.

First, some context. The goal is a fun, small-ish systems programming language for when you can't use a garbage collector.[^2]

Garbage collectors give you a lot. Use-after-free and double free are non-issues. They work with just about any data structure, including graphs. Closures are easy to use. And they can automatically call your deinitializer at the right time, so you don't have to worry about manually cleaning up external resources like file descriptors.

I want Blip to give you as much memory safety as you get with a GC,[^3] which is a big lift. I don't want to add the burden of statically guaranteed data race freedom too – that's well trodden ground in language design, and it adds a lot of complexity. 



[^1]: Be warned: much of the README is out of date, as are other files in the repo, but you might still be interested in poking around.

[^2]: Garbage collectors are great. They work wonderfully for most applications, and I use garbage collected languages whenever I can. But there are some domains, like the firmware I've been writing [at work](https://www.k2space.com), where a GC just isn't practical, and I want something nice for those domains too. 

[^3]: This includes bounds checking, which isn't strictly a benefit of garbage collection, but is table stakes for most new languages. 