---
layout: post
title: To Center A Button
description: "How to center a button that is 50% of its parent?"
tags: [android, button, custom view]
---

Carloss Sessa of [50 Android Hacks](http://www.amazon.com/50-Android-Hacks-Carlos-Sessa/dp/1617290564) starts off with an interesting problem: how to center a button that is 50% of its parent width?

<center>
  <figure>
    <img src="http://i.imgur.com/DW6JWEo.png"/>
    <figcaption>Pretty cool Android phone, huh?</figcaption>
  </figure>
</center>

It's easy to center a generic button. Using <code>FrameLayout</code>, one could simply set the appropriate layout gravity:

{% highlight xml %}
<?xml version="1.0" encoding="utf-8"?>
<FrameLayout xmlns:android="default_schema"
             android:layout_width="match_parent"
             android:layout_height="match_parent">

    <Button android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_gravity="center"
            android:text="Button"/>

</FrameLayout>
{% endhighlight %}

But how to set the button width to 50% of its parent? I decided to create a custom `Button`. 

When creating custom views, you usually want to override some [standard methods](http://developer.android.com/reference/android/view/View.html) the framework uses. There are two I could consider in this case: 

* `onMeasure` determines the size requirements for this view and all of its children
* `onLayout` assigns a size and position to all of its children.

Because the parent layout already provides the position I'm after, `onMeasure` will suffice.

{% highlight java %}
public class CenteredButton extends Button {

    // Constructors ommited for clarity.

    @Override
    protected void onMeasure(int wMeasureSpec, int hMeasureSpec) {
        super.onMeasure(wMeasureSpec, hMeasureSpec);

        final View parent = (View) getParent();
        final int halfParentWidth = parent.getWidth() / 2;

        setMeasuredDimension(halfParentWidth, getMeasuredHeight());
    }
    
}
{% endhighlight %}

The custom `onMeasure` is straightforward: after invoking `super`, which provides initial measurements, we get the parent width, halve it and apply the result. Then, we tell the XML layout to use it:

{% highlight xml %}
<?xml version="1.0" encoding="utf-8"?>
<FrameLayout xmlns:android="default_schema"
             android:layout_width="match_parent"
             android:layout_height="match_parent">

    <path.to.CenteredButton android:layout_width="wrap_content"
                            android:layout_height="wrap_content"
                            android:layout_gravity="center"
                            android:text="Button"/>

</FrameLayout>
{% endhighlight %}

It works! However, there's a slight hiccup - the text is not centered. Applying `android:gravity="center"` doesn't work:

<center>
  <figure>
    <img src="http://i.imgur.com/4huD0rH.png"/>
    <figcaption>"Button" text is not centered.</figcaption>
  </figure>
</center> 

Darn `super.onMeasure`! A closer look at the [source code](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/4.4.2_r1/android/widget/Button.java/) reveals a simple text like "Button" is drawn with the help of a `BoringLayout`. The layout is also responsible for positioning the text based on the available height, width and gravity. 

Since I only change the width after `super.onMeasure`, it has no effect to the initially calculated dimensions and position.

Is there a way to halve the width before `super.onMeasure`? Enter [MeasureSpec](http://developer.android.com/reference/android/view/View.MeasureSpec.html). Scary documentation hides its simple nature: the three modes, `AT_MOST`, `UNSPECIFIED` and `EXACTLY`, are a rough translation of the familiar `wrap_content`, `match_parent` and a specific size, e.g. `200dp`. 

Two encoded values of the `onMeasure` method, `widthMeasureSpec` and `heightMeasureSpec` present the `MeasureSpec` values from its parent View. This is handy, we can easily extract the parent width from it: 

{% highlight java %}
final int parentWidth = MeasureSpec.getSize(widthMeasureSpec);
{% endhighlight %}

If we could modify the `widthMeasureSpec` to be half its parent width, and pass it to the `super.onMeasure`, our work would be done. Fortunately, that's easy:

{% highlight java %}
@Override
protected void onMeasure(int wMeasureSpec, int hMeasureSpec) {
  final int widthSize = MeasureSpec.getSize(wMeasureSpec);
  final int halfWidth = widthSize / 2;
  final int newMeasureSpec = 
    MeasureSpec.makeMeasureSpec(halfWidth, MeasureSpec.EXACTLY);
    
  super.onMeasure(newMeasureSpec, hMeasureSpec);
}
{% endhighlight %}

Mission accomplished! 

Of course, Carlos did things a bit differently. To see his solution, go and buy his book.