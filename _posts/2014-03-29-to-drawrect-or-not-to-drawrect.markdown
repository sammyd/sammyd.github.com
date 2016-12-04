---
layout: post
title: "To drawRect or not to drawRect?"
date: 2014-03-29 09:09
comments: true
tags: iOS, talks, mobile
permalink: "blog/:year/:month/:day/:title"
cover: /assets/covers/all_the_way_down.jpg
navigation: true
---

When you go searching StackOverflow for some instructions on how to do some
custom drawing on iOS, you'll often find that the top answer instructs you to
create a custom subclass of `UIView`, and override the `drawRect:` method. But
is this always the best approach? What does `drawRect:` actually do, and what
are the other options for custom drawing on iOS?

In March 2014 I popped over to the US on behalf of
[ShinobiControls](http://shinobicontrols.com) and presented this as a talk at
[CocoaConf DC](http://cocoaconf.com/dc-2014/home) and
[CocoaConf Mini Austin](http://cocoaconf.com/austin-2014/home). I've collected the
slides and relevant links in this post for you to enjoy.

<!-- more -->


The slides are below - and are available on SpeakerDecK:

<script async class="speakerdeck-embed" data-id="f3625e10996f01313e53426a9381af41" data-ratio="1.77777777777778" src="//speakerdeck.com/assets/embed.js"></script>

The accompanying arrow-drawing project is available on Github at
[github.com/sammyd/iOS-ArrowDrawing](https://github.com/sammyd/iOS-ArrowDrawing).
Feel free to fork it and send me pull requests with any mistakes you find =]

I always appreciate comments/complaints hit me up on twitter -
[@iwantmyrealname](https://twitter.com/iwantmyrealname). And since I'm in
full-flow with the self-promotion, you should go and download a copy of my
__free__ eBook - [iOS7: day by day](https://leanpub.com/ios7daybyday).

That's it. I'm all out


sam
