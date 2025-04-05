---
title: "Guaranteed cleanup without RAII"
date: 2025-03-30T16:21:47-04:00
draft: true
---

I'm working on a new programming language. It's called [Blip](https://github.com/davidbalbert/blip).[^1]

Right now, it's a collection of losely connected ideas that don't fit together. There's certainly no code to run. But I think there's something interesting here, and I want to get feedback.

Some context:

- Small and fun.
- Mostly imperative.
- Low level: pointers, no GC, etc.
- Errors are values. No exceptions.
- No async. Go-style procs and channels.
- Pretty safe – no double free or use-after-free or out of bounds access.

At some point I'll write a larger overview. For now, let's talk cleanup.

In a language with [RAII](https://en.wikipedia.org/wiki/Resource_acquisition_is_initialization), you acquire resources (memory, file handles, network connections, etc.) on initialization and clean them up on deinitialization. Each type has a destructor that's called automatically when a value dies. This way you can't forget to clean up.[^2]

Destructors are surprisingly complicated. The goal is good: make sure you don't forget to clean up. But there are rough edges.

How do you deal with destructors that can error?[^3] Because destructors are called implicitly, there's no place to handle errors. 

What if you have a way to clean up a collection of things that's more efficient that cleaning them up one at a time? E.g. a hypothetical `closeall(2)` system call. You generally can't have multiple destructors, so this isn't usually an option[^4].

What if you want to support types with both sync and async destructors? You could try making destructors polymorphic over async-ness, but that gets complicated quickly.

There is good news: you can use functions. If you clean things up explicitly by calling functions, you can do whatever you want – handle errors, clean up an array of values, pick between sync and async cleanup, call a function multiple times to retry, etc.

But now we're back where we started. If you have to clean up explicitly, you might forget to do it:

```
func foo(m *Mutex) {
    m.lock()

    // ...many lines of code...

    return
    // whoops, forgot to unlock
}
```

Some languages like Zig and Go have a `defer` mechanism that lets you do your cleanup right next to your setup. This makes it easier to catch a missing cleanup in code review, and can guarantee that cleanup happens on every branch.

```
func foo(m *Mutex) {
    m.lock()
    defer m.unlock()

    // do stuff

    if condition {
        // early return, unlock is called here
        return
    }

    // unlock is called here too
}
```

If you integrate `defer` with error handling you can get something like [`errdefer`](https://matklad.github.io/2024/03/21/defer-patterns.html) where you clean up only if there's an error.

But there's a problem. You can still forget to clean up.

I think we can get the best of both worlds. We can do it with linear types.[^5].

If you've used Rust, maybe you've heard of affine types. A value of an affine type can be used either zero or one time, but no more. All Rust types are affine unless they implement `Copy`. Though to make things more practical affine values can still be borrowed many times before they're used or dropped. 

A value of a linear type must be used exactly once. Put another way, linear types must be cleaned up!

```
func foo(m *Mutex) {
    // assume Key is a linear type
    var k Key = m.lock()

    // do stuff
    
    // error: key wasn't used
}
```

If we add `defer`, we now have ergonomic, guaranteed cleanup without RAII.

```
func foo(m *Mutex) {
    var k Key = m.lock()
    defer m.unlock(key)

    // do stuff

    if condition {
        // m is unlocked here
        return
    }

    // and here too
}
```

With `errdefer` (I'd spell it `recover`), you can clean up only if something goes wrong. 

```
// Fd is a linear type

func open(path string) Fd | error

func open2(a, b string) (Fd, Fd) | error {
    fd1, err := open(a)
    if err != nil {
        return err
    }
    recover close(fd1)

    fd2, err := open(b)
    if err != nil {
        // fd1 is closed here
        return err
    }

    // but not here – we can return a linear type. Our caller or one of its ancestors will need to consume these eventually. 
    return fd1, fd2
}
```

There are missing pieces. What would happen if you tried to return the key in the mutex example? You need a way to make sure the mutex outlives the key. Ideally without adding full-fat Rust-style explicit lifetimes and borrow checking. There are options. 

A description of error handling is missing too. I'll get to that in another post. For now it's enough to say that functions that can fail return something akin to the now common `Result` or `Either` types without actually being one.


- default to copyable

**TODO: an aside about treating memory different from other resources**

**TODO: an aside about whether values or types should be linear**
- 

[^1]: Be warned: much of the README is out of date, as are other files in the repo, but you might still be interested in poking around.

[^2]: In C++ you can still forget. If you're using heap allocated values created with `new`, it's still easy to forget to `delete`. OTOH, in languages like Rust and Swift, your destructor is almost always called. You have to work very hard to keep that from happening.

[^3]: Before you say this isn't necessary, remember that `close(2)` can error.

[^4]: Well, [maybe you can](https://www.sandordargo.com/blog/2021/06/16/multiple-destructors-with-cpp-concepts), but I'm not sure how far this gets you. It doesn't let you clean up a list of things in one shot.

[^5]: I dislike the name "linear types." Like a lot of names from type theory land, it's too navel-gazey for me. The challenge is to take powerful developments from type theory world and make them intuitive. Give them good names, or even better, let people use these tools without even realizing it.
