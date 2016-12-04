---
layout: post
title: "ReactiveCocoa 2.x with Swift"
date: 2014-07-02 09:57:10 +0100
comments: true
tags: [iOS, swift]
permalink: "blog/:year/:month/:day/:title"
cover: /assets/covers/all_the_way_down.jpg
navigation: true
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

{% highlight objc %}
#import <ShinobiCharts/ShinobiChart.h>
#import <SocketRocket/SRWebSocket.h>
#import <ReactiveCocoa/ReactiveCocoa.h>
{% endhighlight %}

It's actually that easy! I love how simple interoperability is at this level.
However, if you try and compile this (with your Podfile created correctly and
pods installed) then you'll run in to some problems within the ReactiveCocoa
source.

# Compiling ReactiveCocoa in a Swift Project

If you try to build a project now, then the compiler will first attempt to compile
your pods - including ReactiveCocoa. Do it. You'll see that it doesn't work - you
get a compiler error around the methods `and`, `or` and `not` on `RACSignal+Operations`.
This is because of a compiler bug, which will hopefully be fixed in a future
release, but until then we can work around it by renaming those methods in the
ReactiveCocoa source.

Find the __RACSignal+Operations.h__ file in the CocoaPods project, and rename
the aforementioned methods to `rac_and`, `rac_or` & `rac_not`. You'll have to
repeat this in the related implementation (`.m`) file as well. You can then find
all the places that use these methods, by attempting a build (there are only about
three places in the RAC source). Fixing each call by changing its name will work.
Note that it might also be possible to do this using Xcode's refactor tools, but
I've not had the most success in the past.

Now your project will build, yay!

# Using generics to improve syntax

One of the things I like about objective-C is the implicit casting available in
the arguments to blocks. By this I mean the following is the signature for a map
function in RAC (defined on `RACStream`):

{% highlight objc  %}
- (instancetype)map:(id (^)(id value))block;
{% endhighlight %}

Which means that when creating a map stage in your pipeline, it would look like
this:

{% highlight objc %}
map:^id(id *value) {
     return value[@"content"];
 }]
{% endhighlight %}

The block returns an `id`, and takes an `id` for the value parameter. This is so
that in objective-C you can build a functional pipeline which can process any
datatypes (since generics don't exist). However, the syntax allows you to specify
(and therefore implicitly cast) these parameters, by defining your block like
this:

{% highlight objc %}
map:^NSString*(NSDictionary *value) {
     return value[@"content"];
 }]
{% endhighlight %}

Although not strictly necessary (since the compiler will allow you to call any
methods on an `id`), it just allows you to have additional type checking at
compile (and writing) time.

And now we move our attention to the world of Swift. The Swift equivalent to `id`
is `AnyObject`, so the map function now looks like this:

{% highlight swift %}
.map({ (value: AnyObject!) -> AnyObject in
  return value["content"]
})
{% endhighlight %}

If you attempt to build this code then (as of beta2) the compiler will crash.
In order to make this work you might think that the following would work:

{% highlight swift %}
.map({ (value: NSDictionary!) -> NSString in
  return value["content"]
})
{% endhighlight %}

However, Swift's type system doesn't like this (with a somewhat cryptic and
misplaced error message). Therefore you need to explicitly cast:

{% highlight swift %}
.map({ (value: AnyObject!) -> AnyObject in
  if let dict = value as? NSDictionary {
    return dict["content"]
  }
  return ""
})
{% endhighlight %}

You have to do this every time you want to call a `map` function, which in my
opinion is a little bit clumsy.

Which brings us to Swift's generic system, and type inference.

### A generic version of `map`

The syntax I'd like to use is:

{% highlight swift %}
.mapAs({ (dict: NSDictionary) -> NSString in
  return dict["content"] as NSString
})
{% endhighlight %}

So how do we go about building this `mapAs()` extension method. Well, extending
a class in Swift is easy:

{% highlight swift %}
extension RACStream {
  func myNewMethod() {
      println("My new method")
  }
}
{% endhighlight %}

We're going to create a generic `mapAs()` method, which includes the explicit
downcasting and the call to the underlying `map()` method:

{% highlight swift %}
func mapAs<T,U: AnyObject>(block: (T) -> U) -> Self {
  return map({(value: AnyObject!) in
    if let casted = value as? T {
      return block(casted)
    }
    return nil
  })
}
{% endhighlight %}

This specifies that the `mapAs` method has 2 generic params - the input and output,
and that there is a requirement that the output be of type `AnyObject`. The closure
we pass to the `mapAs()` method takes the first generic type and returns the second.

All the `mapAs()` method does is call the underlying `map()` method, but performs
the downcasting as appropriate.

We can write a similar method for filter:

{% highlight swift %}
func filterAs<T>(block: (T) -> Bool) -> Self {
  return filter({(value: AnyObject!) in
    if let casted = value as? T {
      return block(casted)
    }
    return false
  })
}
{% endhighlight %}

This obviously can be extended to all the methods on `RACStream`, `RACSignal` etc.

I find that using these generic methods (combined with Swift's type inference),
leads to a much more expressive pipeline:

{% highlight swift %}
wsConnector.messages
  .filterAs({ (dict: NSDictionary) in
      return (dict["type"] as NSString).isEqualToString("unspecified")
    })
  .mapAs({ (dict: NSDictionary) -> NSString in
    return dict["content"] as NSString
    })
  .deliverOn(RACScheduler.mainThreadScheduler())
  .subscribeNextAs({(value: NSString) in
    self.tickerLabel.text = value
    })
{% endhighlight %}

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
