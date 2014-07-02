---
layout: post
title: "ReactiveCocoa 2.x with Swift"
date: 2014-07-02 09:57:10 +0100
comments: true
categories: [ios, swift]
---

I recently wrote a blog post on the ShinobiControls blog about using
ReactiveCocoa with a ShinobiChart. It's great - you should go and
[read it](http://www.shinobicontrols.com/blog/posts/2014/06/24/reactiveshinobi-using-shinobicharts-with-reactivecocoa).
I was also invited to give a talk at [#bristech](http://briste.ch/) around the
same time, and thought that this blog post would make a really interesting topic.
The audience at #bristech is not an iOS audience. Not even mobile-focused. It's
very much a mixed discipline event, with a heavy focus on javascript (lowest
common denominator etc.). Therefore I decided a general talk on functional
reactive programming, with ReactiveCocoa examples would be a great place to go.

One of the things non-Cocoa developers complain about is the somewhat alien
appearance of objective-C. Now, I don't really think this is a valid complaint,
but in the interests of making my talk more accessible, I decided that if the
examples I gave were in Swift then fewer people would be frightened off.

And so begins the great-swiftening. I took the original project which accompanied
the previous blog post, and swiftified it. There were a few things I thought
might be useful to share. This post is the combination of those thoughts.

<!--more-->

# Bridging Headers

Bridging headers are part of the machinery which enables interaction between
swift and objective-C. They're well-documented as part of Apple's
[interoperability guide](https://developer.apple.com/library/prerelease/mac/documentation/Swift/Conceptual/BuildingCocoaApps/MixandMatch.html#//apple_ref/doc/uid/TP40014216-CH10-XID_75).
Essentially, there is a special header inside your project (specified with a
build setting) into which the objective-C headers for the classes you wish to use
with Swift should be collected.

The __ReactiveWikiMonitor__ project uses 3 objective-C libraries:

- ShinobiCharts
- SocketRocket
- ReactiveCocoa

Therefore, the bridging header looks like this:

    #import <ShinobiCharts/ShinobiChart.h>
    #import <SocketRocket/SRWebSocket.h>
    #import <ReactiveCocoa/ReactiveCocoa.h>

It's actually that easy! I love how simple interoperability is at this level.
However, if you try and compile this (with your Podfile created correctly and
pods installed) then you'll run in to some problems within the ReactiveCocoa
source.

# Compiling ReactiveCocoa in a Swift Project



# Using generics to improve syntax


# Conclusion

This is very much an interim piece of work. We can expect RAC3 to be swift-focused,
and so these techniques won't be required. However, they don't just apply to RAC.
Using generics to simplify block arguments is especially helpful when interfacing
with objective-C which uses `id` as a type.

As ever, the code for this is available on the 'swiftify' branch of the
ReactiveShinobi project on my [github](https://github.com/sammyd/ReactiveShinobi/tree/swiftify).
If you don't fancy having to fiddle with the ReactiveCocoa source once you've
pulled it down, there's also a [swiftify_with_pods](https://github.com/sammyd/ReactiveShinobi/tree/swiftify_with_pods)
branch, which includes the source code changes.

sam
