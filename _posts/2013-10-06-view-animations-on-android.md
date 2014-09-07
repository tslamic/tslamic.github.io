---
layout:   post
title:    View Animations On Android
comments: true
---

The Android framework provides two animation systems:
<ul>
	<li>Property animation (introduced in Android 3.0)</li>
	<li>View animation</li>
</ul>
While property animation system is more flexible, offers more features and generally has better performance, the view animation system takes less time to setup and requires less code to write. If it accomplishes everything you need to do, there is no need to use the property animation system.

In this post, we're going to focus on the basics of the latter one.

The view animation system is used to perform <em>tweened</em> animation on views. Tween stands for "in-between" and refers to the creation of successive frames of animation between the first and the last frame.

It can perform a series of simple transformations on a <code>View</code> object, changing:
<ul>
	<li>position</li>
	<li>size</li>
	<li>rotation</li>
	<li>transparency</li>
</ul>
The system lives in the <code>android.view.animation</code> package, with <code>Animation</code>, <code>AnimationSet</code> and <code>Interpolator</code> being the juicy parts.

<code>Animation</code> class is responsible for a single animation. If our animation should change e.g. position _and_ transparency, we'd have to use the <code>AnimationSet</code> class, since it represents a group of animations that should be played together.

<a href="http://cogitolearning.co.uk/?p=1078" target="_blank"><code>Interpolator</code></a> defines the rate of change of an animation. We can specify, for example, how fast something is going to fade out or if the animation should be accelerating, decelerating, etc.

<center>
  <figure>
    <img src="http://i.imgur.com/xVlimsR.png" />
    <figcaption>Rate of change for some interpolators.</figcaption>
  </figure>
</center>

Constructing an `Animation` is simple:
 
```java
Animation fadeOut = new AlphaAnimation(1f, 0f);
fadeOut.setInterpolator(new AccelerateInterpolator());
fadeOut.setDuration(1000);
```

Remember the simple transformations we're able to do: position, size, rotation and transparency? Android provides an implementation for each one, respectively:
<ul>
	<li>TranslateAnimation</li>
	<li>ScaleAnimation</li>
	<li>RotateAnimation</li>
	<li>AlphaAnimation</li>
</ul>
In our example, we're using <code>AlphaAnimation</code> which is used for transparency transformations. The arguments, <code>1f</code> and <code>0f</code>, simply mean we wish the opacity to go from 100% to 0% - we want the view to fade out.

The second line of our example sets the <code>Interpolator</code>. Again, to make things easier for us, the package includes several Interpolator subclasses specifying various speed curves. We've randomly chosen `AccelerateInterpolator`, which tells a transformation to start slow, then speed up. By default, every Animation uses a <code>LinearInterpolator</code>.

Lastly, we set the duration (in milliseconds) for our animation to run.

Animations can be defined by either XML or code. As with layouts, XML file is recommended. Let's convert the code in our example to XML.

Android designates a special folder where XML files specifying animations should live: `res/anim`. We therefore create a file called `fade_out.xml`:

```xml
<?xml version="1.0" encoding="utf-8">
<alpha xmlns:android="http://schemas.android.com/apk/res/android"
    android:interpolator="@interpolator/accelerate_quad" 
    android:fromAlpha="1.0"
    android:toAlpha="0.0"
    android:duration="1000">
```

and put it in the <code>res/anim</code> folder. To execute it, <code>AnimationUtils</code> class provides a method that constructs the Animation:

```java
Context ctx = getContext();
Animation fade = AnimationUtils.loadAnimation(ctx, R.anim.fade_out);
```

Suppose we wish to - besides fading out - shrink our view. Recall that in order to execute two or more Animations at the same time, we have to use <code>AnimationSet</code>. This is as simple as creating an Animation and adding it to the AnimationSet:

```java
// suppose fade is defined as in the code example above

Animation shrink = new ScaleAnimation(1f, 0f, 1f, 0f);
shrink.setDuration(1000);

final AnimationSet set = new AnimationSet(false);
set.add(fade);
set.add(shrink);

// suppose view variable exist, and points to a View instance
view.startAnimation(set);
```

If you wish to execute animations in the set one by one, you can do that by specifying the _start offset_; <code>fade</code> is set to run 1000 milliseconds. If we want <code>shrink</code> to be executed after <code>fade</code>, it needs to wait 1000 millis before executing:

```java
shrink.setStartOffset(1000);
```

Converting everything to XML yields:

```xml
<?xml version="1.0" encoding="utf-8"?>
<set xmlns:android="http://schemas.android.com/apk/res/android">
  <alpha
    android:fromAlpha="1.0"
    android:toAlpha="0.0"
    android:interpolator="@android:interpolator/accelerate_quad"
    android:duration="1000">

  <scale
    android:fromXScale="1.0"
    android:toXScale="0.0"
    android:fromYScale="1.0"
    android:toYScale="0.0"
    android:duration="1000"
    android:startOffset="1000">
</set>
```

To conclude this rudimentary introduction to view animation, another interface is worth mentioning: <code>AnimationListener</code>. Suppose we wish to hide our view after an animation completes. This can be easily achieved using <code>AnimationListener</code>:

```java
Animation fadeOut = new AlphaAnimation(1f, 0f);
fadeOut.setDuration(1000);

fadeOut.setAnimationListener(new Animation.AnimationListener() {
  @Override
  public void onAnimationStart(Animation animation) {}

  @Override
  public void onAnimationRepeat(Animation animation) {}

  @Override
  public void onAnimationEnd(Animation animation) {
    view.setVisibility(View.INVISIBLE);
  }
});

view.startAnimation(fadeOut);
```

If you wish to spice things up by browsing through some actual code, I've created a simple app showing random View animations. You can find it <a href="https://github.com/tslamic/AndroidExamples/tree/master/SimpleAnimationsExample">here</a>.