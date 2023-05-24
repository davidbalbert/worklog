---
title: "Writing a Text Editor"
date: 2023-05-24T12:23:33-04:00
---

I'm writing a text editor. Until I can think of a better name, it's called [Watt](https://github.com/davidbalbert/Watt).

{{< img src="watt.png" class="image" alt="A screenshot of Watt" >}}

Watt is for my own use. I want to have something that I can use as a daily driver, but that's a tall order given that I'm used to having syntax highlighting, auto-complete, debugging, project-wide search, GitHub Copilot, etc. We'll see how far I get.

One way I can simplify is by supporting extensions. Because Watt is just for me, if there's a feature I need, I can build it in.

I use a Mac, so Watt is a Mac app. A GUI editor is more complicated than a CLI editor, but it's what I like, and it'll give me an excuse to learn more about text layout, rendering, and editing. I'm using Core Text, Apple's low-level text layout and rendering API. There are higher level frameworks, most interestingly TextKit 2, but [after spending time with it](https://github.com/davidbalbert/TextKit-2-Playground) last year, I think it's a bit too buggy. Plus, using Core Text lets me learn more about text layout and rendering.

After ~3 weeks of work, Watt can render text into a scroll view at acceptable speed, and supports text selection and caret movement with the mouse. The next step is actually editing text.

I'm hoping to have text editing, including support for system-provided IMEs, done by tomorrow. That would normally be unrealistic, but I can scavange most of the code from last year's TextKit 2 experiments.
