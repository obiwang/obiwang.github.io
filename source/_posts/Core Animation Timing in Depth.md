title: "Core Animation Timing in Depth"
layout: "post"
lang: "en"
date: 2016-05-06 13:30:00
tags:
- Core Animation
categories:
- iOS
permalink: core_animation_timing
---

The Core Animation timing model is described by the `CAMediaTiming` protocol and inherited by the `CAAnimation` class and `CALayer` class. 

The timing model in `CAAnimation` is easy to understand. There is a great visual descritpion [here](http://ronnqvi.st/controlling-animation-timing/) by David Rönnqvist. There is even a [cheat sheet](http://ronnqvi.st/images/CAMediaTiming%20cheat%20sheet.pdf) to help you to get better understanding with different situations.

Let's look at the timing model in `CALayer`.

## Timing in CALayer

### Parent Time vs. Local Time

According to Apple's [CAMediaTiming Protocol Reference](https://developer.apple.com/library/ios/documentation/GraphicsImaging/Reference/CAMediaTiming_protocol/index.html):

> The CAMediaTiming protocol models a hierarchical timing system, with each object describing the mapping of time values from the object's parent to local time.

A layer define a timespace relative to its superlayer, simliar to a relative coordinate space. To convert from parent time $t_p$ to active local time $t$, there is an equation:

\begin{equation}
t = (t_p - beginTime) \times speed + \mathit{timeOffset} \mspace{20mu} (1)
\end{equation}

There are two kinds of local time, `active local time` and `basic local time`. Apple just refer the name without further description. But, for now, you can just forget about basic local time, and consider the active local time is the local time.

### Speed
> Specifies how time is mapped to receiver’s time space from the parent time space.

Let's look back at equation `(1)`. It is a linear function whose slope is the `speed`. When beginTime is 0: 
$$ t = t_p \times speed + \mathit{timeOffset} $$

When $t_p$ is 0, t should be timeOffset. The function graph is illustrated below.

![Figure 1, beginTime is 0](/images/graph1.svg)

Set a layer's speed to 2.0 means the layer's time runs 2 times faster than its superlayer.

### timeOffset vs. beginTime
> The `beginTime` specifies the number of seconds into the duration the animation should start and is scaled to the timespace of the animation's layer. The `timeOffset` specifies an additional offset, but is stated in the local active time. Both values are combined to determine the final starting offset.

The `beginTime` and the `timeOffset` both contribute to the final starting offset. The `beginTime` is affected by the `speed` while the `timeOffset` is not. You can see why in equation `(1)`. The complete function graph looks like below. It is shifted right `beginTime` compare to Figure 1.

![Figure 2](/images/graph2.svg)

### active local time vs. basic local time

Though there is no actual defination of `basic local time`, this part is just my guessing. You can skip it. You don't have to know anything about the `basic local time` since CA has done for you. But it can help you get better understanding of Core Animation. In the [CAMediaTiming Protocol Reference](https://developer.apple.com/library/ios/documentation/GraphicsImaging/Reference/CAMediaTiming_protocol/index.html):
> The conversion from parent time to local time has two stages:
0. Conversion to “active local time”. This includes the point at which the object appears in the parent object's timeline and how fast it plays relative to the parent.
0. Conversion from “active local time” to “basic local time”. The timing model allows for objects to repeat their basic duration multiple times and, optionally, to play backwards before repeating.

We already know stage 1, which is equation `(1)`. And we can see that stage 2 affects animaiton's  repeating and reverse. We can guess that, the `basic local time` should be the time of one iteration of the animation. So that Core Animation can calcuate the animation value at any `active local time`. The local time of an animation with repeating and reverse is illustrated below.

![Figure 3](/images/graph3.svg)

## Timing in Action
The biggest difference is that once you add an `CAAnimation` to a layer, you can't change its property, the animation is readonly. But you can manipulate some of the `CAMediaTiming` properties in the layer. The *some* of the properties are: `beginTime`, `timeOffset`, and `speed`. You can imagine that the animation is a movie, the layer is a player. Once you have created a movie, you can play it by adding to the layer. You can't edit the movie during playing since it is just a player, not a recoreder or something. But you can pause, fast faward, rewind or start from any position.

### How to play an animation
An animation is not played immediately after being added to a layer, it will be scheduled at the next run loop. If the `beginTime` is not specified in the animation object, the layer will set it to the current local time after the animation started. If you want to do something when the animation started, you can set to the animation's delegate property with an NSObject that implements `animationDidStart(anim: CAAnimation)` method. If the layer's speed is set to 0 prior the animation have started, the animation will never start without changing the layer's speed to a nonzero value.

### How to pause an animation
Apple has documented how to [Pausing and Resuming Animations](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/CoreAnimation_guide/AdvancedAnimationTricks/AdvancedAnimationTricks.html#//apple_ref/doc/uid/TP40004514-CH8-SW15). Let's see why it should code this way.

#### Pause the animation

```swift
func pauseLayer(layer:CALayer) {
    let pausedTime = layer.convertTime(CACurrentMediaTime(), fromLayer:nil)
    layer.speed = 0.0
    layer.timeOffset = pausedTime
}
```

1. Get the current local time, this should be done before pausing
2. Set the layer's speed to zero to pause the animation. Since the target property is not controlled by the animation any more, it will switch back to its model value.
3. In order to preserver the animation's value before pausing, set the layer's `timeOffset` to the previous local time, so that it will freeze at that moment.

#### Resume the animation

Accoring to equation `(1)`, we can transform it to:

\begin{equation}
beginTime = t_p - \frac{t - \mathit{timeOffset}}{speed}\ \mspace{20mu} (2)
\end{equation}

```swift
func resumeLayer(layer:CALayer) {
    let pausedTime = layer.timeOffset
    layer.speed = 1.0
    layer.timeOffset = 0.0
    layer.beginTime = 0.0
    let timeSincePause = layer.convertTime(CACurrentMediaTime(), fromLayer:nil) - pausedTime
    layer.beginTime = timeSincePause
}
```

1. Retrieve the local time when paused stored in `timeOffset`
2. Set the layer's speed back to resume the animation.
3. Set the new `beginTime` so that the animation will resume from paused time.
    1. In equation`(2)`, with `timeOffset` is 0 and speed is 1.0, the new `beginTime` should be $t_p - t$
    2. Reset layer's `timeOffset` and `beginTime` so that the `convertTime` will return the *scaled* $t_p$, because there is no offset between local time and parent time.
    3. The $t$ should be the same as the local time when paused, because the local time has not gone any further.

It is not easy to understand what time it is *now* and *then*. Time doesn't stop and wait for anyone. Let's see a more complicated situation.

### How to change the layer's speed during animating

Suppose we have a animating layer with its `speed` at $m$, then change its speed to $n$. What properties should we adjust?

Let's look at equation`(1)` again
\begin{equation}
t = (t_p - beginTime) \times speed + \mathit{timeOffset} \mspace{20mu} (1)
\end{equation}

* At the time speed changes, we are not going to change the animation state, which means the local time are the same between both speed at that time. $t$ is not changed.
* timeOffset is not affecting by the speed. $timeOffset$ is not changed.
* $t_p$ is not changed.
* $speed$ is known.

So the only things to do is to set the new `beginTime`.

![Figure 4](/images/graph4.svg)

Apply speed $m$ and $n$ to equation`(1)` we get two equations($t_b'$ is what we need):
\begin{align}
t = (t_p - t_b) \times m + \mathit{timeOffset} \newline
t = (t_p - t_b') \times n + \mathit{timeOffset}
\end{align}

Combine both we can get this:
\begin{align}
(t_p - t_b) \times m = (t_p - t_b') \times n
\end{align}

So the new `beginTime` $t_b'$ should be:
\begin{align}
t_b' = t_p - (t_p -t_b) \times \frac{m}{n}\
\end{align}

The $t_b$ is simply layer.beginTime. The $t_p$ is a little tricky. We can reset layer's `timeOffset` and `beginTime` to get the *scaled* $t_p$, then divide by the old speed, we will get the real $t_p$.

### Keys to solve timing issue
* Use the equation
* Make sure which changes and which does not
* local time usually represent the current state of animation
* set layer's `timeOffset` or `beginTime` to fast forward or rewind
* reset layer's `timeOffset` and `beginTime` to get the *scaled* parent time