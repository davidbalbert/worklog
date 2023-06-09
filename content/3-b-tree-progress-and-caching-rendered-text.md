---
title: "B-tree progress and caching rendered text"
date: 2023-06-13T10:33:41-04:00
---

After over a week of banging my head against ropes, I finally feel like I'm over the hump, and things are getting easier.

I've been reading through [Xi](https://github.com/xi-editor/xi-editor/tree/master/rust/rope), [swift-collections](https://github.com/apple/swift-collections/tree/main/Sources/RopeModule), and Jesse Grosjean's [SummarizedCollection](https://github.com/jessegrosjean/SummarizedCollection). David Scot Taylor's [YouTube series on B-trees](https://www.youtube.com/watch?v=I22wEC1tTGo) has been another good resource. Volume 3 of The Art Of Computer Programming also has a good description of B-trees on pages 482 - 489, but it's dense, and some of the conventions – like how it defines the tree's order and leaves being virtual and not actually existing – are different from most other materials that I found.

The big question I've been wrestling with, besides "what is a B-tree and how does it work," is "how do I efficiently concatinate two B-trees, where all elements of the left tree are less than all elements of the right tree?" I had a hunch that there was a way to do this in O(log n) time, but the code in Xi and swift-collections was pretty inscrutable, and there aren't many good resources online, probably because B-Trees are usually used for file system and database indexes where I think concatination is a rare operation. As a comparison, the naive way of doing it, by iterating over all the elements of the shorter tree and inserting them into the longer tree, is probably O(m log n), where *m* is the number of elements in the shorter tree, and *n* is the number of elements in the longer one. When both trees are huge, this is untenable.

Most of the libraries I looked at had a different way of doing the concatination. The one that ended up clicking was the way Xi does it, so I'm doing that. I'll probably write up a description of how this works at some point soon.

The other thing that's made this difficult is figuring out how to implement a copy on write collection in Swift. The convention in Swift is that everything, including collections, are passed by value, but multiple copies of a collection can share a reference to the same underlying storage. The storage is only copied when you write to the underlying array.

*Caching rendered text*

One of the mistakes I often make while programming is abstracting too soon. I'll think "let's solve this problem by implementing data structure X," only to find that the interface X provides doesn't actually let me solve the problem. The good news is I know that people have written text editors with ropes before, but I was still a bit nervous about whether I was going to finish building a rope, only to find that I can't solve my problems with it. This morning while walking the dog, I wrote up some notes that convinced me I'm on the right track, and have reproduced them below.

Here's some context that might make them more comprehensible:

A CALayer is part of [Core Animation](https://developer.apple.com/documentation/quartzcore). For our purposes, you can think of it as a texture stored in GPU memory that gets composited into a window.

A CTLine is part of [Core Text](https://developer.apple.com/documentation/coretext). It represents a line of text that's been laid out and can be drawn into a canvas like a CALayer.

A Rope is a data structure for storing large amounts of text. It has O(log n) indexing, insertion, and deletion. Even better, it has O(1) copying because it's immutable by default. Ropes get these properties by storing their data in trees.

Compare this to a standard string stored as a fixed-length array of characters, where all these operations are O(n)[^1].

[In their original construction](https://www.cs.tufts.edu/comp/150FP/archive/hans-boehm/ropes.pdf), ropes were balanced binary trees, where each leaf contained a portion of the full text, and each internal node contained the sum of the lengths of the text stored in its children.

These days, it seems most people build ropes out of B-trees[^2]. B-trees are similar to self-balancing binary search trees, but instead of having only two children, each node can have N children, where N can be very large (e.g. 1024). Unlike AVL trees or red-black trees, which require a  rebalancing step after insertion and end up only partially balanced, B-trees are always perfectly balanced by construction – all leaves are always at the same height, splitting when necessary. Rather than growing down from the leaves, B-trees grow up from the roots.

"Summary" means the data that's stored in each internal node of the rope. By default, that's total length of the text contained in the node's children, but it can be other things too, like the number of newline characters in its children.

Here are my notes from this morning:

> What am I trying to do?
>
> Make the text editor fast.
>
> Things I think are probably slow:
>
> - Drawing CTLines into a CALayer
> - Creating CTLines
> - Calculating line breaks (not actually sure how expensive this is)
>
> I definitely want to cache the CALayers. I think caching line breaks is easy with ropes, so I’ll do it. I’m not sure if it’s necessary to cache CTLines separately from the layers that contain their rasterized contents.
>
> To cache CALayers, I need to be able to invalidate the layers when the text in their paragraph changes, and remove layers that are no longer in preparedContentRect.
>
> The obvious first attempt at invalidating layers when text changes is mapping textRange to layer.
>
> Some problems:
>
> - If the paragraph is edited but doesn’t change length, we won’t invalidate.
> - if the paragraph changes length, we have to update the ranges of all following paragraphs.
>
> I’m pretty sure I can deal with the first problem by doing the same two step edit that Xi does: when performing an edit, generate a set of deltas from the existing rope, and then apply them to create a new rope. We might be able to use these deltas to invalidate the new rope: I.e., an edit that leaves a paragraph the same length is a Replace(text in range) where length(text) = length(range), or alternatively a Delete(range) followed by Insert(text at range.start). Either way, we know that any layers in range should be invalidated from the cache.
>
> I believe the second problem can be solved using a rope: if the summary is the length of each paragraph (rather than the paragraph’s range in the document), we should be able to find the associated layers for a given text range by walking down the tree in O(log n) time, and updates will still be be O(log n) time because lengths (unlike ranges) just require summing as we go up the tree rather than offsetting all the following ranges, which would be O(n).
>
> I’m less certain about only caching what’s in the viewport. We need a method like invalidateLayers(outside: rect). If we store the heights of each paragraph in the cache, we should be able to quickly find the start index and end index of the rect and then get a sub tree containing only the inside.
>
> I think this implies that we can’t have a single rope that stores all the stateful data. We need a separate rope just for the layer cache. We also need to duplicate either the line heights or the paragraph lengths in the layer cache, and we need to store an offset for the beginning of the layer cache that so that offsets in the layer cache (which represents just a portion of the document) are compatible with offsets in the line height cache (which represents the entire document). The reason we can’t store the layer cache in the same rope as the heights and other layout information, is that implementing invalidateLayers(outside: rect) without slicing the tree requires iterating over most of the tree to find the layers to invalidate. That’s O(n) while slicing is O(log n).
>
> For deciding whether to cache the CTLines, the questions are:
>
> - How much time does it take to layout a CTLine?
> - Is there anything besides rendering the text that we need CTLines for?
> - How much memory to CTLines take up?
>
> For the first question, I’ll have to measure. I know text shaping is a complicated process, but I wouldn't be surprised if computers are fast enough that we don't need to cache.
>
> The reason for the second question is that we’ll already be caching the rendered text, so if we only need the CTLines for drawing, and we’re caching the results of drawing, there’s no reason to cache the CTLines too. The only thing I can think that we might need CTLines for is if we wanted to do something like calculating the height of the entire document in the background after loading it (this would give us the actual document height rather than just an estimated height). This is probably not necessary, at least to start. Assuming our font is monospaced, the number of glyphs per line is constant. We should be able to accurately estimate the height of each paragraph by doing glyphsInParagraph/glyphsPerLine × lineHeight.
>
> For the third question, CTLines are pointers to a struct of unknown size, so the only way to figure this out is empirically by creating a ton of them and measuring memory usage. Given the answer to the second question though, I don’t think we’ll need to cache CTLines. Furthermore, even if we did want to calculate an fully accurate document height in the background, we could do that without caching the CTLines – just create one CTLine at a time, ask it for its height, and then throw it away.

[^1]: You might think indexing in normal strings is O(1), but that's only true for fixed-length encodings like ASCII or UTF-32. UTF-8, which is the most common text interchange format, and UTF-16, which is used by Core Text, are both variable-width, which means you need to scan the entire string to get to the nth code point or extended grapheme cluster.

[^2]: Technically they're probably more like B+trees, which only store values in the leaves, not in the intermediate nodes, but B-trees and B+trees are both key-value stores, and ropes store, so neither analogy is totally accurate.
