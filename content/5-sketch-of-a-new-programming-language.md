---
title: "Sketching out a new programming language"
date: 2025-03-30T16:21:47-04:00
draft: true
---

I'm working on a new programming language. It's called [Blip](https://github.com/davidbalbert/blip).[^1]

Right now, it's a collection of losely connected ideas that don't fit together. There's certainly no code to run. But I think there's something interesting here, and I want to get some feedback.

At some point I'll write a bigger overview. But for today, let's stick with the most interesting ideas I've found so far: safe mutable aliasing, and guaranteed cleanup without RAII.


## Goals

- A fun, small-ish systems programming language for when you can't use a GC.
- Spacial and temporal safety. The same things you'd get from a GC.
- Guaranteed resource cleanup
- A fast, helpful compiler. I don't want to fight my tools.

Fun is subjective, but to me, Go feels fun. I like the syntax. I like the names it uses. Pointers are fun.

## Non-goals

- Zero-cost abstractions. [Some good abstractions have costs](https://graydon2.dreamwidth.org/307291.html) and I want to 

[^1]: Be warned: much of the README is out of date, as are other files in the repo, but you might still be interested in poking around.
