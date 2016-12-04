---
layout: post
title: "A faster array in objective-c"
date: 2012-09-29 15:50
comments: true
tags: [obj-c, computer vision, iOS] 
---

This post was originally planned to be about how I've adapted the pointer
arithmetic used with standard data a fairly standard data structure to work
well with `NSMutableData` in objective-C. However, in making a sample project
to demonstrate this, I've expanded it a little to speeding up a very specific
type of array in objective-C.

## Motivation in image processing

I have been working recently on some image processing algorithms, one of which
required a priority queue of pixel locations. Now, this essentially requires a
list of integers, but since images have lots of pixels, it's a very long list.
`NSArray` is a really useful all-purpose collection class used an awful lot
in objective-C. The one 'restriction' is that it is a collection of objects - 
it's not possible to collect primitives in an NSArray without wrapping them in
an object. `NSNumber` is an object which represents all the different number
types and, more often than not, wrapping your numbers in this is the best way
to go. However, I mentioned that I want to build a list of a large number of
integers (think tens of millions). At this point, the overhead of wrapping
integers in `NSNumber` to put them in an `NSArray` becomes significant.

In the past, when doing image processing work, it is at this point I would
drop down to C (from python, matlab, ruby etc) and use low-level memory management
functionality to exactly create the data structures as expected. This is also
an option when using objective-C - it's just a superset of C, so `malloc()` and
it's associated functionality are available to use. However, in using this
we loose all of the memory management functionality made available by
objective-C. There is another way though - `NSMutableData`.


## NSMutableData as a chunk of memory

`NSMutableData` is a class which provides the user with a writeable contiguous block of
memory of a given size. It inherits from `NSObject` so the memory management
of reference counting and autorelease pools comes for free. It also has the
advantage that we can ask for the block of memory to be dynamically resized and
iOS will take care of this for us. There is one proviso with this - and that is
that iOS reserves the right to move our block of data (primarily when resize
is requested). This makes perfect sense, but does cause some issues with
standard pointer-based data structures.

In this post I'll describe how to implement a basic linked-list of integers using
`NSMutableData` and compare its performance to that of an `NSMutableArray`
based list.

<!-- more -->

## Linked lists

Linked lists are one of the simplest pointer-based data structures. It is a
collection of nodes, each of which contains some data and a pointer to where
you can find the next element in the list. I'm not going to talk much more about
them - checkout [Wikipedia](http://en.wikipedia.org/wiki/Linked_list) - it knows all.

{% highlight objc %}
typedef struct Node
{
    Node *nextNode;
    int   value;
} Node;
{% endhighlight %}

As I mentioned before, there is an issue when using pointers with `NSMutableData` - 
in that you cannot guarantee that your block of data won't be moved around.
Therefore, instead of using a pointer to the next node, we record the offset.
Whenever the block of memory is relocated, the pointers of each node will change,
but their relative offset from the front of the block will remain the same:

{% highlight objc %}
typedef struct Node
{
    int nextNodeOffset;
    int value;
} Node;
{% endhighlight %}


In order to demonstrate this process with a toy project, I defined a pretty simple
protocol which my dynamically-sized arrays should implement:

{% highlight objc %}
#import <Foundation/Foundation.h>

@protocol DynamicSizedArray <NSObject>

@required
- (id)initWithCapacity:(int)capacity;

- (void)pushBack:(int)p;
- (void)pushFront:(int)p;

- (int)popBack;
- (int)popFront;

@end
{% endhighlight %}

### Code highlights

I'm not going to reproduce the entirety of the code within this
post, but I've put together a sample project on GitHub to demonstrate
it, so you can pull it down from there. It's at
[github.com/sammyd/LinkedList-NSMutableData](https://github.com/sammyd/LinkedList-NSMutableData).

At initialisation time we create a cache of nodes of the correct
size and then initialise it:

{% highlight objc %}
int bytesRequired = capacity * sizeof(Node);
nodeCache = [[NSMutableData alloc] initWithLength:bytesRequired];
[self initialiseNodesAtOffset:0 count:capacity];
{% endhighlight %}

Every time we further extend the nodeCache then we'll need to initialise
the newly created nodes, so have pulled that out into another method:

{% highlight objc %}
- (void)initialiseNodesAtOffset:(int)offset count:(int)count
{
    Node *node = (Node *)nodeCache.mutableBytes + offset;
    for (int i=0; i<count - 1; i++) {
        node->value = 0;
        node->nextNodeOffset = offset + i + 1;
        node++;
    }
    node->value = 0;
    // Set the next node offset to make sure we don't continue
    node->nextNodeOffset = FINAL_NODE_OFFSET;
}
{% endhighlight %}


Pushing a new value into the array is pretty simple:

{% highlight objc %}
- (void)pushFront:(int)p
{
    Node *node = [self getNextFreeNode];
    node->value = p;
    node->nextNodeOffset = topNodeOffset;
    topNodeOffset = [self offsetOfNode:node];
}
{% endhighlight %}

Pushing to the end of the array is pretty similar - both use a method
which gets them the next free node:

{% highlight objc %}
- (Node *)getNextFreeNode
{
    if(freeNodeOffset < 0) {
        // Need to extend the size of the nodeCache
        int currentSize = nodeCache.length / sizeof(Node);
        [nodeCache increaseLengthBy:_cacheSizeIncrements * sizeof(Node)];
        // Set these new nodes to be the free ones
        [self initialiseNodesAtOffset:currentSize count:_cacheSizeIncrements];
        freeNodeOffset = currentSize;
    }
    
    Node *node = (Node*)nodeCache.mutableBytes + freeNodeOffset;
    freeNodeOffset = node->nextNodeOffset;
    return node;
}
{% endhighlight %}

This method is the one responsible for resizing the nodeCache if required.
`NSMutableData` has the method to `increaseLengthBy:`, it's just a matter
of setting them as empty nodes and resetting the free node location. It
is at this point that pointer-based addressing would fail since the memory
block is likely to move location. We could implement a method which loops
through all the existing nodes and updates their pointers, but the offset
addressing seems cleaner and works just as well.

And finally, the last method required by the protocol is popping nodes:

{% highlight objc %}
- (int)popFront
{
    if(topNodeOffset == FINAL_NODE_OFFSET) {
        return INVALID_NODE_CONTENT;
    }
    
    Node *node = [self nodeAtOffset:topNodeOffset];
    int thisNodeOffset = topNodeOffset;
    
    // Remove this node from the queue
    topNodeOffset = node->nextNodeOffset;
    int value = node->value;
    
    // Reset it and add it to the free node cache
    node->value = 0;
    node->nextNodeOffset = freeNodeOffset;
    freeNodeOffset = thisNodeOffset;
    
    return value;
}
{% endhighlight %}

Here we grab the first node, move the 'pointer' to the first node to the
second node, move the old first node to the available nodes cache and return
the value.


## Testing

Obviously, this kind of code is perfect for some unit testing. I've written
the bare bones of a test suite - enough to iron out one or two bugs I came
across. I'm not going to bore you with the tests here in this blog, but they're
in [github](https://github.com/sammyd/LinkedList-NSMutableData).


## NSArray Implementation

As a comparison, I have made an implementation which uses `NSMutableArray`. The
salient parts are below:

{% highlight objc %}
- (void)pushFront:(int)p
{
    [array insertObject:[NSNumber numberWithInt:p] atIndex:0];
}

- (int)popFront
{
    int v;
    if(array.count > 0) {
        v = [[array objectAtIndex:0] intValue];
        [array removeObjectAtIndex:0];
    } else {
        v = INVALID_NODE_CONTENT;
    }
    return v;
}
{% endhighlight %}


## Profiling the two approaches

The original purpose behind this work was dealing with large numbers of
integers in an array, so to compare the two approaches we'll see how long
it takes to push 10 million integers into the array and then popping
them off again:

{% highlight objc %}
- (NSArray*)runListProfileWithList:(id<DynamicSizedArray>)list maxSize:(int)maxSize
{    
    double startTime = CACurrentMediaTime();
    for (int i=0; i < maxSize; i++) {
        [list pushFront:arc4random() % 1000000];
    }
    double pushTime = CACurrentMediaTime();
    
    int poppedValue = 0;
    while (poppedValue != INVALID_NODE_CONTENT) {
        poppedValue = [list popFront];
    }
    
    double popTime = CACurrentMediaTime();
    
    return [NSArray arrayWithObjects:[NSNumber numberWithDouble:(pushTime - startTime)],
                                     [NSNumber numberWithDouble:(popTime - pushTime)],
                                     nil];
}
{% endhighlight %}

The demo app in the github repo allows the user to run this once with each
implementation and displays the results. The following is a screen shot from running it
on the simulator on my ageing MacBook.

![](/images/2012-09-29-profile-screen.png)

As you can see, for 10 million integers, the linked list implementation is significantly
faster - over 3 times faster in fact. Since the integers don't have to be wrapped in `NSNumber`s
the memory footprint is also smaller.


## Conclusion

So, I've managed to create a basic array implementation which is faster than `NSArray`
for primitive data types. I'm not suggesting that it should replace `NSArray` - far from
it. For 99% of cases, `NSArray` is likely to be the best choice. But if you've got
a large number of primitive types you need to put in an array, then it might be worth
considering using `NSMutableData` and building your own implementation.

Use the demo implementation on [github](https://github.com/sammyd/LinkedList-NSMutableData)
at your own risk. I built it to discover whether significant performance gains are possible
- it's not necessarily production-ready ;)

sx
