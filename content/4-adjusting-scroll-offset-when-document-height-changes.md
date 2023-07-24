---
title: "Adjusting scroll offset when document height changes"
date: 2023-07-24T10:33:41-04:00
---

In an NSScrollView – and likely in scroll views in other systems – if the height of your document changes, you may need to adjust your scroll offset to compensate so that the content you see in your viewport doesn’t jump around.

Here are some ways your document height can change:


- Laying out a paragraph that wasn’t previously laid out.
- Changing the font size or zoom level.
- Reloading the document due to a change on disk.
- Editing the document. This is multifaceted. Consider:
	- Multiple cursors.
	- Two views (windows, tabs, etc.) onto the same document.
	- Edits by someone else in a collaborative editor.

This is a surprisingly complex problem, and there’s no one-size fits all solution.

Different types of documents may call for different rules: in a drawing app, when you zoom, you may want to maintain the center point in your view. That is to say, what was at the center before the zoom should remain at the center afterwards.

In a text editor, you probably want to maintain the position of the upper left corner. But even this is more complicated than it first appears. If you’re zooming or resizing text, that will cause each paragraph to reflow, and text that was at the beginning of each line fragment (each visual line in a wrapped paragraph) will change. How do you even determine what the “same location” is?

From here on out, we’ll just be considering text editors.

There are two variables to pay attention to: the height of the document itself, and the scroll offset – how far down in the document we’re scrolled, usually measured from the upper left corner of your viewport (the part of the document you can see).

NSScrollView will not change the content offset in response to a change in document height. If the document height grows, your content offset will remain the same. This means the percent of the document you’ve scrolled through will go down. If you were half way through the document before, you may now only be 45% of the way through. If the document height shrinks, everything will be reversed.

It’s not enough to know that the height of the document has changed. You need to know where the height changed within the document.

If there’s a change in document height that occurs below your viewport, you won’t experience any ill effects. If the document grows, the scroll nub will get smaller and move up towards the top of the scroll bar, but there will be no visual glitches in the viewport. If the document is tall enough relative to the window size, the changes in the scroll bar may not even be visible.

If the change in document height occurs above your viewport, you will notice a glitch. Specifically, if the document height grows, it will be as if you had instantaneously jumped up towards the top of the document. The change is jarring.

The fix is to adjust the content offset by the amount that the height changed. If the document got 10px bigger, and it happened above your viewport, you need to increase the content offset by 10px, and then it will look like nothing changed. Easy.

The implementation details are a bit more complicated. When laying out a text document, the unit of work is generally a paragraph (i.e. the text between two newline characters). If your in a programmer’s text editor, you might call this a line even though it takes up multiple visual lines on the screen. This means the most basic info you might have is “the height of paragraph 23 changed from 50px to 60px”.

But even that isn’t enough to figure out if you need to adjust the content offset. If the bounding box of the paragraph is entirely above the viewport (`paragraph.maxY <= viewport.minY`), then it's easy. Apply the adjustment.

If the paragraph contains `viewport.minY`, the question becomes: in which line fragment in the paragraph did the height change occur? If the line fragment is above the viewport, you need to apply the adjustment, but if it's inside the viewport you shouldn't. I thought you might have to specially deal with a line fragment that partially overlaps (i.e. contains `viewport.minY`), but it turns out that's not necessary – if you can see any portion of the line fragment that caused the height change, you don't apply an adjustment.

In short, if the height change occurred visually within the viewport or below it, don’t do anything. If the height change occurred above the viewport, apply the adjustment.

If you’re doing contiguous layout – laying out the document starting at the top all the way down through the viewport – scrolling up generally shouldn’t cause any layout, and your document height won’t change. But contiguous layout is expensive, especially for large documents.

Instead, you might do non-contiguous (i.e. viewport-based) layout. That is to say, only laying out the text that you can see in the viewport. This is a lot faster, but it means you don’t actually know the height of the document[^1], so you have to estimate. As you move the viewport and layout paragraphs, you can replace the estimated height of each paragraph with the actual height. This is how you can get in a situation where scrolling up will cause your document height to change: grab the scroll bar and quickly scroll to the bottom, missing some paragraphs, and then start scrolling up with your trackpad.

With collaborative editing, things get even weirder. Let’s say you have two carets in your viewport. The upper one is your friend’s, who’s editing the document remotely, and the lower one is yours. With the rules we have so far, if your friend starts typing, we won’t apply any adjustment. Every time your friend types enough to add another line fragment, everything below it will get pushed down, including your caret.

But we probably don’t want this! If you’re in the middle of writing, it would be pretty annoying to have your caret slowly move down the page while other people make edits. So here’s another rule: if you’re in a collaborative editing context, your carets are privileged over other people’s carets. If someone is typing into your viewport, and your caret is in the viewport too, we should anchor to the top of your caret instead of anchoring to the top of the viewport. That way, as your friend adds new line fragments, the paragraph they’re editing will seem to grow upwards, rather than down, and your caret will stay in the same location. To do this, we have to apply the adjustment any time a paragraph above your in-viewport caret is changed by someone else’s caret, even if the affected line fragment is inside the viewport.

Of course, if you don’t have a caret in the viewport, we go back to the original rules.

I don’t actually know if this is a good idea because I’ve never seen it in action. Furthermore, what about other types of changes in the viewport above your caret that don’t come from collaborative editing? I’m not sure! I’m not even sure what might cause those changes, but it’s worth thinking about.

There’s one more collaborative editing wrinkle: what if you’re in the same situation as above, but instead of a caret in the viewport, you have a selection? In that case, I think we should probably go back to the previous rule: your friend typing in the viewport will move your selection down with the rest of the text. I’m not even sure why I think this, and I could be wrong, but my gut says it’s right. Carets are more important than selections.

The good news is that if you’re not making a collaborative editor (I’m not), you can sidestep some of these issues. But having multiple cursors or having the same document open in multiple tabs, where each tab has its own scroll state, carets, etc., has many of the same complexities as collaborative editing – though I don’t think there’s anything analogous to our rule about your caret being more important than theirs.


[^1]: Technically, with contiguous layout you might not know the height of the document either, but you will know the exact height of the portion of the document that starts at the top and ends at the bottom of your viewport, which is enough to deal with the jumping when you scroll up.
