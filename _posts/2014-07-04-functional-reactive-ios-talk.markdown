---
layout: post
title: "Functional Reactive iOS: Talk"
date: 2014-07-04 10:20:31 +0100
comments: true
tags: [iOS, talks, frp]
permalink: "blog/:year/:month/:day/:title"
cover: /assets/covers/all_the_way_down.jpg
navigation: true
---

All too often we as developers spend our lives working out how to get data from
one part of our program to another. And then dealing with side effects associated
with shared state. These are a couple of the problems that the functional
reactive programming paradigm can help to solve.

In July 2014, I presented a talk at [#bristech](http://briste.ch/) which looked
at what exactly functional reactive programming is, and how it can help you with
building your own applications.

<!-- more -->

The __iOS__ part of the title is a bit of a misnomer - although my examples are
all from the world of CocoaTouch, there is not much which is specifically related
to __ReactiveCocoa__.

The slides are available:

<script async class="speakerdeck-embed" data-id="7412d090e4fc0131e1bf4ab20097e045" data-ratio="1.77777777777778" src="//speakerdeck.com/assets/embed.js"></script>

The code for the two sample projects is available on github:

- [github.com/sammyd/RACTextFieldEventLog](https://github.com/sammyd/RACTextFieldEventLog)
- [github.com/sammyd/ReactiveShinobi](https://github.com/sammyd/ReactiveShinobi) (if you are interested in the Swift
  version, look at the __swiftify__ or __swiftify_with_pods__ branches)

If you are interested in more technical detail about the project itself, then I
wrote a [blog post](http://www.shinobicontrols.com/blog/posts/2014/06/24/reactiveshinobi-using-shinobicharts-with-reactivecocoa)
on the [ShinobiControls](http://www.shinobicontrols.com/) blog,
explaining how to link [ReactiveCocoa](https://github.com/ReactiveCocoa/ReactiveCocoa)
with a ShinobiControls chart. I have also written an [article](/blog/2014/07/02/reactivecocoa-2-dot-x-with-swift/)
about how to use Swift with RAC 2.x, and the power that the type inference and
generics affords you.

I shall add the video to this page once it has arrived on the internets.

Feel free to gimme a shout on twitter if you have any questions / comments - I'm
[@iwantmyrealname](https://twitter.com/iwantmyrealname).


sam
x
