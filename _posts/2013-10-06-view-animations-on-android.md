---
layout: post
title: View Animations On Android
description: "How to do view animations on Android"
tags: [android, animation, interpolator]
---

The Android framework provides two animation systems:

 - Property animation (introduced in Android 3.0)
 - View animation

While property animation system is more flexible, offers more features and generally has better performance, the view animation system takes less time to setup and requires less code to write. If it accomplishes everything you need to do, there is no need to use the property animation system.

In this post, we're going to focus on the basics of the latter one.

The view animation system is used to perform _tweened_ animation on views. Tween stands for "in-between" and refers to the creation of successive frames of animation between the first and the last frame.

It can perform a series of simple transformations on a `View` object, changing:

 - position
 - size
 - rotation
 - transparency

The system lives in the `android.view.animation` package, with `Animation`, `AnimationSet` and `Interpolator` being the juicy parts.

`Animation` class is responsible for a single animation. If our animation should change e.g. position _and_ transparency, we'd have to use the `AnimationSet` class, since it represents a group of animations that should be played together.

`Interpolator` defines the rate of change of an animation. We can specify, for example, how fast something is going to fade out or if the animation should be accelerating, decelerating, etc.

<center>
  <figure>
    <a href="http://cogitolearning.co.uk/?p=1078" target="_blank">
      <img src="http://i.imgur.com/xVlimsR.png" />
    </a>
    <figcaption>Rate of change for some interpolators.</figcaption>
  </figure>
</center>

Constructing an `Animation` is simple:
 
{% highlight java %}
Animation fadeOut = new AlphaAnimation(1, 0);
fadeOut.setInterpolator(new AccelerateInterpolator());
fadeOut.setDuration(1000);
{% endhighlight %}

Remember the simple transformations we're able to do: position, size, rotation and transparency? Android provides an implementation for each one, respectively:

 - `TranslateAnimation`
 - `ScaleAnimation`
 - `RotateAnimation`
 - `AlphaAnimation`

In our example, we're using `AlphaAnimation` which is used for transparency transformations. The arguments, `1` and `0`, simply mean we wish the opacity to go from 100% to 0% - we want the view to fade out.

The second line of our example sets the `Interpolator`. Again, to make things easier for us, the package includes several subclasses specifying various speed curves. We've randomly chosen `AccelerateInterpolator`, which tells a transformation to start slow, then speed up. By default, every Animation uses a `LinearInterpolator`. Lastly, we set the millisecond duration for our animation to run.

Animations can be defined by either code or XML. Android designates a special folder where XML files should live: `res/anim`. Let's create a file called `res/anim/fade_out.xml`:

{% highlight xml %}
<?xml version="1.0" encoding="utf-8"?>
<alpha xmlns:android="http://schemas.android.com/apk/res/android"
    android:interpolator="@interpolator/accelerate_quad" 
    android:fromAlpha="1.0"
    android:toAlpha="0.0"
    android:duration="1000"/>
{% endhighlight %}

To use it, `AnimationUtils class provides a method that constructs the Animation:

{% highlight java %}
Context ctx = getContext();
Animation fade = AnimationUtils.loadAnimation(ctx, R.anim.fade_out);
{% endhighlight %}

Suppose we wish to - besides fading out - shrink our view. Recall that in order to execute two or more animations at the same time, we have to use `AnimationSet`. This is as simple as creating an `Animation` and adding it to the `AnimationSet`:

{% highlight java %}
// suppose fade is defined as in the code example above

Animation shrink = new ScaleAnimation(1, 0, 1, 0);
shrink.setDuration(1000);

final AnimationSet set = new AnimationSet(false);
set.add(fade);
set.add(shrink);

// suppose view variable exist, and points to a View instance
view.startAnimation(set);
{% endhighlight %}

If you do not wish to run all animations in the set at the same time, you can specifying the _start offset_. `fade` will run 1000 milliseconds. In order for `shrink` to be executed afterwards, it needs to wait 1000 millis:

{% highlight java %}
shrink.setStartOffset(1000);
{% endhighlight %}

Converting everything to XML yields:

{% highlight xml %}
<?xml version="1.0" encoding="utf-8"?>
<set xmlns:android="http://schemas.android.com/apk/res/android">
  <alpha
    android:fromAlpha="1.0"
    android:toAlpha="0.0"
    android:interpolator="@android:interpolator/accelerate_quad"
    android:duration="1000"/>

  <scale
    android:fromXScale="1.0"
    android:toXScale="0.0"
    android:fromYScale="1.0"
    android:toYScale="0.0"
    android:duration="1000"
    android:startOffset="1000"/>
</set>
{% endhighlight %}

To conclude this rudimentary introduction to view animation, an interface is worth mentioning: `AnimationListener`. Suppose we wish to hide our view after an animation completes or just listen to animation events. All we have to do is:

{% highlight java %}
Animation fadeOut = new AlphaAnimation(1f, 0f);
fadeOut.setDuration(1000);

fadeOut.setAnimationListener(new Animation.AnimationListener() {
  @Override
  public void onAnimationStart(Animation animation) {
    // Do something when fadeOut starts
  }

  @Override
  public void onAnimationRepeat(Animation animation) {
    // Do something if fadeOut is repeated
  }

  @Override
  public void onAnimationEnd(Animation animation) {
    view.setVisibility(View.INVISIBLE);
  }
});

view.startAnimation(fadeOut);
{% endhighlight %}

If you wish to spice things up by browsing through some actual code, I've created a simple app showing random View animations. You can find it [here](https://github.com/tslamic/AndroidExamples/tree/master/SimpleAnimationsExample).