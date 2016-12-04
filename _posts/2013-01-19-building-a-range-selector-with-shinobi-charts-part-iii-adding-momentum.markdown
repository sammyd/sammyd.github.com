---
layout: post
title: "Building a range selector with ShinobiCharts: Part III - Adding momentum"
date: 2013-01-19 21:32
comments: true
published: true
tags: [iOS, shinobi]
permalink: "blog/:year/:month/:day/:title"
cover: /assets/covers/all_the_way_down.jpg
navigation: true
---

> This tutorial is also available on the [ShinobiControls](http://www.shinobicontrols.com/blog/posts/2013/04/09/building-a-range-selector-with-shinobicharts-part-iii)
blog. You'll find better support and assistance on this site as part of
[ShinobiDeveloper](http://www.shinobicontrols.com/shinobideveloper)

This is the third post in a series about creating a range selector using Shinobi
charts for iOS. If you haven't already read the previous posts
([part I](/blog/2013/01/11/building-a-range-selector-with-shinobi-charts-part-i-linking-2-charts/),
[part II](/blog/2013/01/15/building-a-range-selector-with-shinobi-charts-part-ii-creating-custom-handle-annotations/))
I reckon that thisone will make a lot more sense if you do.

The code is available on github at
[github.com/sammyd/Shinobi-RangeSelector](https://github.com/sammyd/Shinobi-RangeSelector), and
combined with a copy of ShinobiCharts (or a 30-day demo) from
[shinobicontrols.com](http://www.shinobicontrols.com/) you can get the entire
project up and running pretty quickly.

At this point in the project we've managed to create 2 charts, one of which
allows the user to interact with the data in the way we'd expect with an iOS chart,
and the other of which has a range selection annotation, which demonstrates which
section of the entire dataset the user is currently viewing. The user is able to
interact with the range selector to change the bounds of the main chart's view,
as well as the location.

We left off last time with a bug (not really the best practice, but the post was
getting a bit on the long side), which would allow a user to drag the upper range
boundary below the lower:

![](/images/2013-01-19-range-selector-broken.png)

Let's start off by fixing that, and then we'll move on to looking at the altogether
more sexy problem of adding momentum to the range selector's motion. Let's stop
waffling and get coding...


<!-- more -->

## Minimum allowable span

It wouldn't be a difficult fix to just prevent a user from dragging the grippers
over the top of each other, but a much more elegant solution would be to have a
minimum span, below which the chart cannot be zoomed. This is useful for general
usage in ShinobiCharts - not just for a range selector. For example, if you have
data which you know is spaced one-per-day, then it doesn't make sense for a user
to be able to zoom in to a range of 10 seconds - we'd like to set a minimum span
of say 1 week.

We need to address this issue in 2 places - one when the user interacts with the
main chart, and one with the range selector. We'll start by looking at the main
chart.

### Main chart interaction

We'll add an ivar to `ShinobiRangeSelector` which will contain the minimum allowed
span value:

{% highlight objc %}
@interface ShinobiRangeSelector () <ShinobiRangeAnnotationDelegate> {
    ...
    CGFloat minimumSpan;
}
@end
{% endhighlight %}

For now, we just set this in the constructor. It might make more sense to pull
this out as a property later on.

{% highlight objc %}
- (id)initWithFrame:(CGRect)frame datasource:(id<SChartDatasource>)datasource splitProportion:(CGFloat)proportion
{
    self = [super initWithFrame:frame];
    if (self) {
    	...
        // Set a minimum span of 4 days
        minimumSpan = 3600 * 24 * 4;
        ...
    }
    return self;
}
{% endhighlight %}

In order to prevent the main chart from zooming below this range we can update the
`sChartIsZooming:withChartMovementInformation:` delegate method implementation
to check the range and reset it if it is smaller than our allowed range:

{% highlight objc %}
- (void)sChartIsZooming:(ShinobiChart *)chart withChartMovementInformation:(const SChartMovementInformation *)information
{
    // We need to check that we haven't gone outside of our allowed span
    if ([chart.xAxis.axisRange.span floatValue] < minimumSpan) {
        // Re-zoom it
        CGFloat midValue = [chart.xAxis.axisRange.span floatValue] / 2 + [chart.xAxis.axisRange.minimum floatValue];
        CGFloat newMin = midValue - minimumSpan / 2;
        CGFloat newMax = midValue + minimumSpan / 2;
        [chart.xAxis setRangeWithMinimum:@(newMin) andMaximum:@(newMax)];
    }
    [rangeAnnotationManager moveRangeSelectorToRange:chart.xAxis.axisRange];
}
{% endhighlight %}

Here we check what the span is, and if it is smaller, then reset the span to the
minimum allowed, whilst maintaining the centre value.


### Range selector

We've now fixed it so that we can't zoom in more than a specified amount by
interacting with the main chart, but it's still possible to use the handles on
the range selector to bypass this.

We'll start by adding a new constructor to the `ShinobiRangeAnnotationManager` to
pass in the minimum range, and an ivar to store it:

{% highlight objc %}
@interface ShinobiRangeAnnotationManager : NSObject
...
- (id)initWithChart:(ShinobiChart *)chart minimumSpan:(CGFloat)minSpan;
@end
{% endhighlight %}

{% highlight objc %}
@interface ShinobiRangeAnnotationManager ()<UIGestureRecognizerDelegate> {
    ...
    CGFloat minimumSpan;
}
@end

@implementation ShinobiRangeAnnotationManager
- (id)initWithChart:(ShinobiChart *)_chart
{
    return [self initWithChart:_chart minimumSpan:3600*24];
}

- (id)initWithChart:(ShinobiChart *)_chart minimumSpan:(CGFloat)minSpan
{
    self = [super init];
    if(self) {
        chart = _chart;
        minimumSpan = minSpan;
        [self createAnnotations];
        [self prepareGestureRecognisers];
    }
    return self;
}
...
{% endhighlight %}

Notice that we keep our previous constructor, and chain them together, adding a
default value.

Now the only time we actually need to check that we haven't broken this minimum
span restriction is when we're dragging the handles on either side of the range
selector. This is all handled within the `handleGripperPan:` method, and so we
just need to update it to only allow the range to be updated if it doesn't
violate this restriction:

{% highlight objc %}
- (void)handleGripperPan:(UIPanGestureRecognizer*)recogniser
{
    CGPoint currentTouchPoint = [recogniser locationInView:chart.canvas];
    
    // What's the new location we've dragged the handle to?
    double newValue = [[chart.xAxis estimateDataValueForPixelValue:currentTouchPoint.x] doubleValue];
    
    SChartRange *newRange;
    // Update the range with the new value according to which handle we dragged
    if(recogniser.view == leftGripper) {
        // Left handle => change the range minimum
        // Check bounds
        if([rightGripper.xValue floatValue] - newValue < minimumSpan) {
            newValue = [rightGripper.xValue floatValue] - minimumSpan;
        }
        newRange = [[SChartRange alloc] initWithMinimum:@(newValue) andMaximum:rightGripper.xValue];
    } else {
        // Right handle => change the range maximum
        // Check bounds
        if(newValue - [leftGripper.xValue floatValue] < minimumSpan) {
            newValue = [leftGripper.xValue floatValue] + minimumSpan;
        }
        newRange = [[SChartRange alloc] initWithMinimum:leftGripper.xValue andMaximum:@(newValue)];
    }
    
    // Move the selector
    [self moveRangeSelectorToRange:newRange];
    
    // And fire the delegate method
    [self callRangeDidMoveDelegateWithRange:newRange];
}
{% endhighlight %}

You can see that we've just added 2 conditional sections to this method - one for
each gripper. We check that we aren't trying to make the range too small, and if
we are then simply reset it to the minimum range. This has the effect that as the
user drags the gripper it will appear to stop moving once the minimum span has
been reached. As they then drag back in the opposite direction, the range will
expand again, as expected.

In order to wire this up correctly, we just need to use the new constructor when
we create the annotation manager in `ShinobiRangeSelector`:

{% highlight objc %}
- (void)createRangeChartWithFrame:(CGRect)frame
{
    ...
    // Add some annotations
    rangeAnnotationManager = [[ShinobiRangeAnnotationManager alloc] initWithChart:rangeChart minimumSpan:minimumSpan];
    rangeAnnotationManager.delegate = self;
    ...
}
{% endhighlight %}


## Range selector momentum

The other task I want to address in this post is adding momentum to the range
selector's draggable motion. This means that when you let go of it, it shouldn't
just stop dead, but should decelerate gracefully like dragging the main chart does.
Since we're using anything which provides this, we're going to roll our own
momentum animation - but don't worry - it's not as difficult as it sounds!

The way in which we wish the momentum animation to work is at the moment which
the user releases the range selector annotation from dragging, it should continue
to move in the same direction, with an appropriate deceleration curve. It's easy
to find when a given gesture has been completed, so we simply need to write the
animation code.

### MomentumAnimation utility class

We'll create a utility class which will allow linear momentum animations. We'll
aim to make this suitably generic, so create a simple class with one method:

{% highlight objc %}
@interface MomentumAnimation : NSObject
- (void)animateWithStartPosition:(CGFloat)startPosition
	               startVelocity:(CGFloat)velocity
	                    duration:(CGFloat)duration
	              animationCurve:(SChartAnimationCurve)curve
	                 updateBlock:(void (^)(CGFloat))updateBlock;

@end
{% endhighlight %}

Let's break this down into the different parameters:

- `startPosition`: Since we're creating a generic utility class, we're going to
use normalised distance - in the range of [0,1]. In our particular scenario, we
will use 0 and 1 to represent the extrema on the range chart, and we'll calculate
the value using the touch location at the instant the pan gesture is completed.
- `startVelocity`: In order to get the get great user experience, we need to take
into account the speed with which the user is dragging when they let go of the
selector. If they user is dragging really slowly then the range selector should
travel less far than if they are dragging quickly - this is the conservation
of momentum. A pan gesture recogniser provides a velocity vector, but since our
animation is one dimensional, we only need a one dimensional velocity, with the
sign representing the direction.
- `duration`: How long the animation should last in seconds.
- `animationCurve`: We'll get to this in more detail later on, but this determines
what shape the velocity-time curve should take. These are provided as utilities
by ShinobiCharts, and include decay, acceleration, linear and ease in/out.
- `updateBlock`: Since we're making a generic animation class, it won't know how
to update the position in order to perform the animation. Therefore we provide a
block to allow the user to specify how to update positions. This block takes one
argument - a normalised position (i.e. in the same scale as the `startPosition`
parameter). As an alternative, we could define a delegate protocol, but I think
that a block is a bit cleaner for this use case.

Note that in the implementation in the repo, we also provide some other
animation methods which provide default values for some of these parameters. 

In the corresponding implementation for the animation method we save off some
ivars and define some additional ivars:

- `animating`: This boolean states whether or not the animation is currently
active. We'll need this later on so that we can cancel animations should we
wish to.
- `startPos` and `endPos`: The start and end positions for the animation. These
are calculated from the provided `startPosition`, `velocity` and `duration`
arguments. The equation for calculating the `startPos` is somewhat empirical - 
it all comes down to what 'feels right' when a user interacts with the app.
Note that we fix the positions to the [0,1] range we defined as our domain.

{% highlight objc %}
@interface MomentumAnimation () {
    CGFloat animationStartTime, animationDuration;
    void (^positionUpdateBlock)(CGFloat);
    CGFloat startPos, endPos;
    BOOL animating;
    SChartAnimationCurve animationCurve;
}
@end

@implementation MomentumAnimation
- (void)animateWithStartPosition:(CGFloat)startPosition
			       startVelocity:(CGFloat)velocity
			            duration:(CGFloat)duration
			      animationCurve:(SChartAnimationCurve)curve
			         updateBlock:(void (^)(CGFloat))updateBlock
{
    /*
     Calculate the end position. The positions we are dealing with are proportions
     and as such are limited to the range [0,1]. The sign of the velocity is used
     to calculate the direction of the motion, and the magnitude represents how
     far we should expect to travel.
    */
    endPos = startPosition + (velocity * duration) / 5;
    
    // Fix to the limits
    if (endPos < 0) {
        endPos = 0;
    }
    if (endPos > 1) {
        endPos = 1;
    }
    
    // Save off the required variables as ivars
    positionUpdateBlock = updateBlock;
    startPos = startPosition;
    
    // Start an animation loop
    animationStartTime = CACurrentMediaTime();
    animationDuration = duration;
    animationCurve = curve;
    animating = YES;
    [self animationRecursion];
}
{% endhighlight %}

The only other thing the API animation method does is to set some animation
values - the animation start time, and the animating boolean. It then calls the
`animationRecursion` method, the naming of which should give some idea as to how
we are going to perform the animation.

{% highlight objc %}
- (void)animationRecursion
{
    if (CACurrentMediaTime() > animationStartTime + animationDuration) {
        // We've finished the alloted animation time. Stop animating
        animating = NO;
    }
    
    if (animating) {
        // Let's update the position
        CGFloat currentTemporalProportion = (CACurrentMediaTime() - animationStartTime) / animationDuration;
        CGFloat currentSpatialProportion = [SChartAnimationCurveEvaluator evaluateCurve:animationCurve atPosition:currentTemporalProportion];
        CGFloat currentPosition = (endPos - startPos) * currentSpatialProportion + startPos;
        
        // Call the block which will perform the repositioning
        positionUpdateBlock(currentPosition);
        
        // Recurse. We aim here for 20 updates per second.
        [self performSelector:@selector(animationRecursion) withObject:nil afterDelay:0.05f];
    }
}
{% endhighlight %}

Let's walk through what this method does:

1. Firstly we check whether the animation should have been completed - i.e. has
the specified time passed (`duration`) since the animation first began? If it has
then we should update the `animating` ivar accordingly.
2. If the `animating` ivar is `NO` then we drop out of the end of this method - 
animation completed. Otherwise we continue.
3. We need to update the position - this is where the aforementioned animation
curve comes into play. Shinobi provides a set of pre-defined animation curves,
and a class which can 'evaluate' them. Evaluation of a curve accepts a normalised
time value, and returns a normalised distance - i.e. we provide a curve type and
the proportion of the curve completed (in the temporal domain) and we will get
back a spatial proportion. We calculate the temporal completion proportion from
the current time, the start time and the duration. From this we get a spatial
proportion, which we need to map to the normalised space the momentum animation
is using. We do this with a simple linear mapping.
4. We now need to actually update the position of the object we are moving, which
we do using the block. As specified, the block takes one variable - the normalised
distance we've just calculated. When we use the class we will define this block
ourselves.
5. Finally we need to recurse - i.e. call ourselves again after a given time, so
that the position will be incrementally updated. Here we use a standard `NSObject`
method to delay a message send a given amount of time. Here we've gone with a
delay of 0.05s, which will represent a framerate of up to 20fps. We won't get this
in reality, but the animation looks smooth enough at this rate.


### Using the MomentumAnumation class

Now that we've gone to the effort of creating the `MomentumAnimation` class, we
should integrate it into the range selector code itself. We'll create one
reusable instance of the animation class:

{% highlight objc %}
@interface ShinobiRangeAnnotationManager ()<UIGestureRecognizerDelegate> {
	...
    MomentumAnimation *momentumAnimation;
}
@end

@implementation ShinobiRangeAnnotationManager
- (id)initWithChart:(ShinobiChart *)_chart minimumSpan:(CGFloat)minSpan
{
    self = [super init];
    if(self) {
        ...
        // Let's make an animation instance here. We'll use this whenever we need momentum
        momentumAnimation = [MomentumAnimation new];
    }
    return self;
}
@end
{% endhighlight %}

The only place we want to use the animation is when the use stops dragging the
range annotation, so we only need to update the `handlePan:` method:

{% highlight objc %}
- (void)handlePan:(UIPanGestureRecognizer*)recogniser
{
    // What's the pixel location of the touch?
    CGPoint currentTouchPoint = [recogniser locationInView:chart.canvas];
    
    if (recogniser.state == UIGestureRecognizerStateEnded) {
        // Work out some values required for the animation
        // startPosition is normalised so in range [0,1]
        CGFloat startPosition = currentTouchPoint.x / chart.canvas.bounds.size.width;
        // startVelocity should be normalised as well
        CGFloat startVelocity = [recogniser velocityInView:chart.canvas].x / chart.canvas.bounds.size.width;

        // Use the momentum animator instance we have to start animating the annotation
        [momentumAnimation animateWithStartPosition:startPosition
                                      startVelocity:startVelocity
                                           duration:1.f
                                     animationCurve:SChartAnimationCurveEaseOut
                                        updateBlock:^(CGFloat position) {
            // This is the code which will get called to update the position
            CGFloat centrePixelLocation = position * chart.canvas.bounds.size.width;
            
            // Create the range
            SChartRange *updatedRange = [self rangeCentredOnPixelValue:centrePixelLocation];
            
            // Move the annotation to the correct location
            [self moveRangeSelectorToRange:updatedRange];
            
            // And fire the delegate method
            [self callRangeDidMoveDelegateWithRange:updatedRange];
        }];
        
    } else {                
        // Create the range
        SChartRange *updatedRange = [self rangeCentredOnPixelValue:currentTouchPoint.x];
        
        // Move the annotation to the correct location
        [self moveRangeSelectorToRange:updatedRange];
        
        // And fire the delegate method
        [self callRangeDidMoveDelegateWithRange:updatedRange];
    }
}
{% endhighlight %}

Although this looks complicated, we haven't really changed all that much from
the original implementation. We now check the state property of the gesture
recogniser - if the gesture has completed (`UIGestureRecognizerStateEnded`) then
we kick off the animation, otherwise, we do exactly as we did before.

In order to start the animation we need to normalise the position and the velocity,
which we do by dividing their pixel values by the width of the range chart's
canvas. Then we invoke the animation method on the momentum animation object, 
passing the expected arguments. We've used `SChartAnimationCurveEaseOut` here as
that represents a pleasant deceleration. The block we pass in to update the
range selector position works as follows:

1. We calculate the new pixel location of the centre - this is multiplying the
normalised position by the width of the chart's canvas.
2. Then we use the utility method to calculate the new required range.
3. This range is passed to the `moveRangeSelectorToRange:` method to update
the position of the
4. And then finally we call the delegate method to make sure that the main chart
is updated as well.

We've done all of these things before, in response to the direct user interaction
passed by the gesture recogniser. Here we are just replacing that with the
animation - really quite simple!

If you fire up the app now and play with it you'll see that the momentum works
really rather well. Try dragging the range selector along at different speeds
and letting go - you'll see the scrolling with momentum as we wanted.


### Interacting with an animating property

As ever, there's a situation in which there is a problem. Once an animation starts
it will continue to update the position until the duration time has been completed.
If you attempt to interact with either the main chart, or the range selector whilst
this animation is happening, then the result with be a strange flickering, as
two different processes attempt to control the position of a single object
simultaneously. In order to fix this problem, we will provide a way of stopping
a currently running animation.

We add a simple method to the API of `MomentumAnimation`:

{% highlight objc %}
@interface MomentumAnimation : NSObject
...
- (void)stopAnimation;
@end
{% endhighlight %}

Since we have the conditional check in the animation recursion method, stopping
the animation is really simple - we just have to set the `animating` ivar to
`NO`. Then on the next recursive call, we'll just drop out of the loop:
{% highlight MomentumAnimation.m objc %}
@implementation MomentumAnimation
- (void)stopAnimation
{
    animating = NO;
}
@end
{% endhighlight %}

So when do we need to stop the animation? Well, it should be stopped every time
we change the range selector's range, except those when we are animating.
We're going to change the `moveRangeSelectorToRange:` method to include this
animation stopping functionality:

{% highlight objc %}
- (void)moveRangeSelectorToRange:(SChartRange *)range cancelAnimation:(BOOL)cancelAnimation
{
    if (cancelAnimation) {
        // In many cases we want to prevent the animation fighting with the UI
        [momentumAnimation stopAnimation];
    }
    
    // Update the positions of all the individual components which make up the
    // range annotation
    leftLine.xValue = range.minimum;
    rightLine.xValue = range.maximum;
    leftShading.xValue = chart.xAxis.axisRange.minimum;
    leftShading.xValueMax = range.minimum;
    rightShading.xValue = range.maximum;
    rightShading.xValueMax = chart.xAxis.axisRange.maximum;
    leftGripper.xValue = range.minimum;
    rightGripper.xValue = range.maximum;
    rangeSelection.xValue = range.minimum;
    rangeSelection.xValueMax = range.maximum;
    
    // And finally redraw the chart
    [chart redrawChart];
}

- (void)moveRangeSelectorToRange:(SChartRange *)range
{
    // By default we'll cancel animations
    [self moveRangeSelectorToRange:range cancelAnimation:YES];
}
{% endhighlight %}

We add the `cancelAnimation:` argument, which, if specified to be `YES` will send
the momentum animation ivar a `stopAnimation` method. The rest of the method
updates the annotation values as we were doing before.

We update the `moveRangeSelectorToRange:` method to call this new method with
`cancelAnimation` set to `YES`. This means that all the places we have used this
API method will now cancel animation before they try and update the position of
the range selector. This is fine and dandy for all but one place - in the position
update block for the animation itself. If we cancel the animation whilst animating
then it will never actually animate. Therefore we update the `updatePosition`
block as follows:

{% highlight objc %}
- (void)handlePan:(UIPanGestureRecognizer*)recogniser
{
	...
    // Use the momentum animator instance we have to start animating the annotation
    [momentumAnimation animateWithStartPosition:startPosition
                                  startVelocity:startVelocity
                                       duration:1.f
                                 animationCurve:SChartAnimationCurveEaseOut
                                    updateBlock:^(CGFloat position) {
        ...
        // Move the annotation to the correct location
        // We use the internal method so we don't kill the momentum animator
        [self moveRangeSelectorToRange:updatedRange cancelAnimation:NO];
        ...
    }];
    ...
}
{% endhighlight %}

Cool. Now if you run up the app again, then you will no longer get the jerky
motion when you try and interact during the momentum animation.


## Onwards

So we've now added a minimum span to the range selector and momentum animation
for when the user is dragging it. We've pretty much got all the really cool
features which are in the 'impress' chart on ShinobiPlay - but there are a couple
of things to take a look at in the next post:

- When we first start the app, we don't have a nice default range. We'll look
at how to set this.
- The other feature we'd like to add is the value annotation on the main chart.
This takes the form of a horizontal line which tracks the y-value of the right-
most visible point on the chart, along with a text label which specifies its
value.

### Update 2013/03/10

Removed use of internal ShinobiCharts methods in line with the code in the
repository.

