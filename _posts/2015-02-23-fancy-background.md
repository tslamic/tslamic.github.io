---
layout: post
title: "Fancy Background"
description: "Efficiently animate resource Drawables"
tags: [android, bitmap, memory, cache, subsampling]
---

FancyBackground is a tiny Android library designed to animate images on a View instance.

Before | After 
:-----------:|:-----------:
![plain](http://i.imgur.com/7kH0FIN.png?1) | ![fancybg](http://i.imgur.com/Sh4XegD.gif)

Android runs on a variety of devices, with different screen sizes and densities. Providing alternative resources is a standard practice. Let’s assume that in our case, the design team only gave us resources of `xxhdpi` size. To make matters worse, an `OutOfMemoryError` is thrown when we try to load any image into memory on our S3 mini. 

> Given that you are working with limited memory, ideally you only want to load a lower resolution version in memory. The lower resolution version should match the size of the UI component that displays it. An image with a higher resolution does not provide any visible benefit, but still takes up precious memory and incurs additional performance overhead due to additional on the fly scaling.

is what the [docs](http://developer.android.com/training/displaying-bitmaps/load-bitmap.html) say about efficient bitmap loading. Therefore, in order to load the background images into memory successfully, we have to subsample and, most likely, scale them. 

This real-world scenario is the motivation behind FancyBackground. FancyBackground animates through a set of resource Drawables. It ensures they are subsampled and cached, if need be, with all the heavy lifting done in the background. The usage is simple. To animate between `R.drawable.fst`, `R.drawable.snd` and `R.drawable.trd`, showing each for 2.5 seconds, simply do:

{% highlight java %}
FancyBackground.on(view)
               .set(R.drawable.fst, R.drawable.snd, R.drawable.trd)
               .inAnimation(R.anim.fade_in)
               .outAnimation(R.anim.fade_out)
               .interval(2500)
               .start(); 
{% endhighlight %}

The code above gives us subsampling, caching and automatic transitions for free. 

There are other options available, so let's do a quick abstract:

Method name | Description
:-----------|:-----------
`set` | sets the Drawable resources we wish to show/animate
`inAnimation` | specifies the animation used to animate a `View` entering the screen.
`outAnimation` | specifies the animation used to animate a `View` exiting the screen.
`loop` | continuously loop through the Drawables or stop after the first cycle is complete.
`interval` | the millisecond interval a Drawable instance will be displayed for.
`scale` | determines how the Drawables should be resized or moved to match the size of the view we're animating on.
`listener` | receives the `FancyBackground` events (described below)
`cache` | caches loaded bitmaps so we don't have to do it again

`FancyListener` can receive four events: 

- `onStarted` when the FancyBackground is started 
- `onNew` when a new image is set
- `onLoopDone` if looping is set to false and the first cycle is complete
- `onStopped` when the FancyBackground stops.

`FancyCache` enables you to create your own bitmap cache. `FancyLruCache` is the default, targeting ~25% of the available heap and evicting the least recently used bitmap if over capacity. Use `null` to avoid caching.

To try it out, make sure to add the following to your dependency list: 

{% highlight bash %}
compile 'com.github.tslamic.fancybackground:library:1.0'
{% endhighlight %}

For an example, visit my [Github profile](https://github.com/tslamic/FancyBackground). 