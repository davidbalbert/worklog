---
title: "Getting tied up with ropes"
date: 2023-06-07T10:34:41-04:00
---

I got text input working a while ago. Unsurprisingly, it didn't take a day, like I estimated. More like three or four.

The big problem is that it's slow. The reason is because of the way I'm doing document height estimation.

You need to know the height of your document to correctly render your scroll bars. Most generally, to know the height of a text document, you need to lay it out from top to bottom. But that's expensive, especially for large documents. It's cheaper to only layout the text you're looking at, but then you don't know your document height, and you're back to square one.

To get around this, you can estimate the height of the document. Leaving out some of the details: I'm storing an array of heights, y-offsets, and character ranges for each line in the document. Each line starts with an arbitrary height estimate of 14 points, and I update the estimates to the actual height when I layout a line.

I'm arbitrarily estimating the height of each line to be 14 points, and then updating estimates as I layout the text. This is plenty fast, even though I have to update all of the following y-offsets every time I layout a line.

When I edit the text however, I have to update all of the following text ranges. This is slow enough that there's noticable lag with each character typed, and all the slowness comes from `String.index(_:offsetBy:)` which takes an opaque `String.Index` and offsets it by an integer value. This is the case even though all my offset values are small – from the beginning of a line to the end of it. From some basic profiling, it seems that the majority of time is spent in `[NSBigMutableStringIndex characterAtIndex:]`, specifically in some locking functions (`os_unfair_lock_lock_with_options` and `os_unfair_lock_unlock`). I don't really know why this is, and it's somewhat frustrating. In general, I find string processing in Swift to be opaque and frustrating.

My height estimation code is also unprincipled and hard to work with. The three or four days I spent on text input were almost entirely spent on updating height estimates correctly.

So armed with these problems, I ran head first into what is a probably bad idea: implementing a [rope](https://www.cs.tufts.edu/comp/150FP/archive/hans-boehm/ropes.pdf). A rope stores a string in a balanced search tree, and is able to do most operations including insertion and deletion in O(log n) time. [Even better](https://xi-editor.io/docs/rope_science_00.html), you can store metadata (e.g. number of line breaks, estimated line heights, etc.) along with the actual characters of your string, and keep them up to date in O(log n) time as well. A year ago, I spoke with [Raph Levien](https://raphlinus.github.io) about this and he specifically recommended against using a rope because they're complicated and hard to implement. But here I am.

For the past week, I have been banging my head against ropes and B-trees with little success. I need to change something about how I'm approaching this problem, but I'm not sure what yet. All I know is that reading books, papers and other people's source code in an undirected way doesn't seem to be working. Hopefully I'll be able to find a better approach.
