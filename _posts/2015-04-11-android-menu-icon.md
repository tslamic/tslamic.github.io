---
layout: post
title: "Android menu icon, delightfully"
description: "Starting a redesign of my app, a contemporary menu to back-arrow icon - like the one introduced in Material design, would come handy. I thought recreating it from scratch would be a great how-to exercise. Here's my recipe."
tags: [android, ui, drawable]
---

There's probably not a single Android developer in the wild that hasn't heard about Material design yet. 

One of my favorite elements are subtle icon transitions, like the ones presented in [delightful details][1].

<div class="center-align">
<video width="320" height="240" loop="true" autoplay="autoplay" controls>
<source src="//material-design.storage.googleapis.com/publish/v_2/material_ext_publish/0B2wX4iIvu8L6ZHZfV1NfRHdCZHM/animation-delightfuldetails-030401_Status_Change_xhdpi_003.webm" type="video/webm">
</video>
</div>

Starting a redesign of my app, a menu to back-arrow icon - like the one in the top left corner - would come handy. I thought recreating it from scratch would be a great how-to exercise. 

> TL;DR Here's the [GitHub repo](https://github.com/tslamic/DelightfulMenuDrawable).

<a name="sec_1"/> 

### A bird's eye view

Menu to back-arrow icon is a simple drawable with transition animation. Both _states_ consist of three sole lines. The easiest way to define the transition is to mark the points of interest: 

<figure class="center-align">
	<img src="http://i.imgur.com/VezpDYs.png" title="mapping" />
	<figcaption>Points will help us define the transition.</figcaption>
</figure>

Intuitively, the mapping is:

- **A to S**
- **B to U**
- C to T
- D to U
- **E to V**
- **F to U**

Points C and D are roughly equal to T and U, so we'll only focus on the bold ones.

### Measures

To extract the width and height of lines in both states, we inspect them in its final form and determine relative lengths compared to the drawable bounds:

<figure class="center-align">
	<img src="http://i.imgur.com/6tgjmsm.png" title="measures" />
	<figcaption>Measuring agains the bounds.</figcaption>
</figure>

At _hdpi_ density, the estimates are:

- $l$ = 60% of bounds.width
- $g$ = 15% of bounds.height
- $w$ = 5% of bounds.height
- $m$ = l/2
- $t$ =  45 deg

Rememeber the variable names, as they will be used throughout this post.

### Progress

To get a smooth transition from e.g. A to S, we have to know the inbetween point at each animation frame. But first, we have to decide how to track animation progress. 

Assuming we'll use property animation, it's on us to provide a property and set of values we want to animate between. 

Let _progress_ denote an arbitrary float value between 0.0 and 1.0, inclusive. This will be our property. We'll start the animation by executing

{% highlight java %}
ObjectAnimator o = ObjectAnimator.ofFloat(drawable, "progress", 0f, 1f);
o.setDuration(animationDuration)
o.start();
{% endhighlight %}

If progress is 0, we show the menu icon. Else if progress is 1, we show the back-arrow icon. 

Progressing from 0 to 1 transforms a home icon to a back-arrow. Conversely, 1 to 0 goes from a back-arrow to home.

<a name="sec_4"/> 

### Math

Remember, the points we refer to are all defined in the <a href="#sec_1">upper section</a>.

<figure class="center-align">
	<img src="http://i.imgur.com/3n4SaAz.png" title="A to S"/>
	<figcaption>Pic 1</figcaption>
</figure>

P is the point at progress $0 < q < 1$, moving between A and S. We want to know $dx$ and $dy$. 

Let's find coordinates of S first, assuming A is the origin. 
Clearly its $x$ coordinate is $m$. To determine $n$, a bigger picture helps:

<figure class="center-align">
	<img src="http://i.imgur.com/Zq5PZ5D.png" title="Bigger A to S" />
	<figcaption>Pic 2</figcaption>
</figure>

Assume $z = g + n$ and $h$ the length between S and D. Recall $t = 45 \deg$. Using trigonometric ratios, we know

$$\sin t = \frac{z}{h}$$

We can get $h$ from

$$
\begin{align}
\cos t &= \frac{m}{h}\\
\frac{\sqrt 2}{2} &= \frac{m}{h}\\
h &= m\sqrt 2\\
\end{align}
$$

Since $h$ is of the same length as the diagonal of a square with an edge $m$, we deduct $z=m$, thus $n = (m-g)$ and $S=(m, (m-g))$. 

Observe that in pic 1, at progress $p=0$, we must be at A. When $p=1$, we must be at S. It's clear that at progress $q$

$$
\begin{align}
dx &= qm \\
dy &= q(m-g)
\end{align}
$$

Another movement we have to take care of is B to D:

<figure class="center-align">
	<img src="http://i.imgur.com/TheualZ.png" title="B to D" />
	<figcaption>Pic 3</figcaption>
</figure>

Only $dg$ is changing. Applying similar logic as above, we know

$$
dg = qg
$$

Note that C-D line (or T-U) is a mirror line. As such, reflecting the $y$ coordinate gets us E-F line changes for free.

> To read more about the approach we've taken, visit the [linear interpolation][2] wiki page.

### Draw

As mentioned, both drawable states have only three simple lines. `Canvas` has a convenient method `drawLine`, which is all we need. 

To be able to draw, we need to calculate `dx`,  `dy` and `dv` and know points A, B, C, D, E, F. 

First three change when progress changes, but others just need bounds. We can compute them in a helper method, which is only invoked when the bounds are set.

{% highlight java %}
private void measure() {
	final Rect bounds = getBounds();
	final float w = bounds.width();
	final float h = bounds.height();

	final float lineLength = LINE_LENGTH * w;
	mHalfLineLength = lineLength / 2;

	mLineGap = LINE_GAP * h;

	final float strokeWidth = LINE_WIDTH * h;
	mPaint.setStrokeWidth(strokeWidth);

	mCenterY = w / 2;
	mStartX = bounds.left + (w - lineLength) / 2;
	mEndX = mStartX + lineLength;
	mTopY = mCenterY - mLineGap;
	mBottomY = mCenterY + mLineGap;

	invalidateSelf();
}
{% endhighlight %}

In the code above, `mHalfLineLength` denotes $m$, `mLineGap` denotes $g$ and `strokeWidth` is $w$. Other instance variables fully determine the points: 

- A(`mStartX`, `mTopY`)
- B(`mEndX`, `mTopY`)
- C(`mStartX`, `mCenterY`)
- D(`mEndX`, `mCenterY`)
- E(`mStartX`, `mBottomY`)
- F(`mEndX`, `mBottomY`)

Each time progress is updated, we invalidate the drawable. Consequently, `draw` method is invoked

{% highlight java %}
 @Override
public void draw(Canvas canvas) {
	final float dx = mProgress * mHalfLineLength;
	final float dy = mProgress * (mHalfLineLength - mLineGap);
	final float dg = mProgress * mLineGap;

	canvas.drawLine(mStartX + dx, mTopY - dy, mEndX, mTopY + dg, mPaint);
	canvas.drawLine(mStartX, mCenterY, mEndX, mCenterY, mPaint);
	canvas.drawLine(mStartX + dx, mBottomY + dy, mEndX, mBottomY - dg, mPaint);
}
{% endhighlight %}

Note that we apply the transformations computed in the <a href="#sec_4">math section</a>. 

### Demo

Because I'm nice, I've wrapped everything we've done so far in a widget for you to try out:

<div class="center-align">
	<div>
	    <canvas id="canvas" width="300" height="300" style="border:1px solid #000000;" />
	</div>
	<div>0.0
	    <input id="slider" type="range" min="0.0" max="1.0" step="0.01" value="0.0" width="300" />1.0
	    <p id="prog"></p>
	</div>
</div>

### Class 4 Strategic Theatre Emergency

If progress is 1, the above widget reveals the pointy end of the arrow is not really pointy.

When drawing on a `Canvas` instance, Android uses [Skia](https://code.google.com/p/skia/), a 2D graphics library, to do the actual drawing. `Canvas.drawLine` invokes a native method `SkCanvas::drawLine`, which invokes `SkCanvas::onDrawPoints`. Because our `mPaint` instance is not complex (it has no path effects, for example), `SkPaint::doComputeFastBounds` is called. 

We care because in this case, if the `mPaint` stroke width is $w$, the rendered point has a radius of $w/2$. 

Here's a zoomed in look at U, where B and F meet and progress is 1:

<figure class="center-align">
	<img src="http://i.imgur.com/4pGRr4J.png" title="source: imgur.com" />
	<figcaption>
	Line A-B (S-U) is colored purple. Line E-F (V-U) is colored green. Red square is the issue we're trying to resolve. </figcaption>
</figure>

If we offset B and F by $(zx, -zy), (zx, zy)$ respectively, we cover the red square. 

Because of the above analysis, we know $r=\frac{w}{2}$. Both lines come at a 45 degree angle, hence $k = 45\deg$ too. We can use trigonometric ratios again:

$$
\begin{align}
zx &= h\cos k \\
zy &= h\sin k
\end{align}
$$

Because $\cos 45 = \sin 45 = \frac{\sqrt 2}{2}$ and $h= \frac{w}{2}$, we get

$$
zx = \frac{w}{2} \cdot \frac{\sqrt 2}{2} = zy = \frac{\sqrt 2}{4}w
$$

Applying progress and updating the `draw` method, we get

{% highlight java %}
@Override
public void draw(Canvas canvas) {
	final float dx = mProgress * mHalfLineLength;
	final float dy = mProgress * (mHalfLineLength - mLineGap);
	final float dv = mProgress * mLineGap;

	// zx/zy computation can be easily moved to the measure method
	final float dz = mProgress * ((Math.sqrt(2)/4)*mPaint.getStrokeWidth());

	canvas.drawLine(mStartX + dx, mTopY - dy, mEndX + dz, mTopY + dv + dz, mPaint);
	canvas.drawLine(mStartX, mCenterY, mEndX, mCenterY, mPaint);
	canvas.drawLine(mStartX + dx, mBottomY + dy, mEndX + dz, mBottomY - dv - dz, mPaint);
}
{% endhighlight %}


### Again, demo

Confirm the fix in previous section resolves the issue:

<div class="center-align">
	<div>
	    <canvas id="canvas2" width="300" height="300" style="border:1px solid #000000;" />
	</div>
	<div>0.0
	    <input id="slider2" type="range" min="0.0" max="1.0" step="0.01" value="0.0" width="300" />1.0
	    <p id="prog2"></p>
	</div>
</div>

### A final touch

To finish off, we have to add rotation. The one in [delightful details][1] goes from:

- 0 to 180, if progress goes from 0 to 1
- -180 to 0, if progress goes from 1 to 0

Because, at this point, you are a master of linear interpolation, the updated `draw` method makes sense

{% highlight java %}
@Override
public void draw(Canvas canvas) {
	final float dx = mProgress * mHalfLineLength;
	final float dy = mProgress * (mHalfLineLength - mLineGap);
	final float dg = mProgress * mLineGap;

	// dx/dy computation was moved to measure method
	final float dz = mProgress * mArrowFactor;

	// mIsBack is true if the icon is currently showing the back arrow
	final float rotationTarget = mIsBack ? -180 : 180;
	final float rotation = mProgress * rotationTarget;
	final float pivotX = getBounds.centerX();
	final float pivotY = getBounds().centerY();

	canvas.save();
	canvas.rotate(rotation, pivotX, pivotY);
	canvas.drawLine(mStartX + dx, mTopY - dy, mEndX + dz, mTopY + dg + dz, mPaint);
	canvas.drawLine(mStartX, mCenterY, mEndX, mCenterY, mPaint);
	canvas.drawLine(mStartX + dx, mBottomY + dy, mEndX + dz, mBottomY - dg - dz, mPaint);
	canvas.restore();
}
{% endhighlight %}

### Parting words

Good on ya for making it this far. I hope math section was clear enough, and the widgets helped. I'm sure some parts can be improved, so I'll be grateful for any feedback I get. The full implementation is [available on GitHub](https://github.com/tslamic/DelightfulMenuDrawable).

<script type="text/x-mathjax-config">
	MathJax.Hub.Config({
		tex2jax: {inlineMath: [['$','$'], ['\\(','\\)']]}
	});
</script>

<script type="text/javascript"
	src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML">

	var DelightfulDrawable = {};

	DelightfulDrawable.newInstance = function (canvasId, sliderId, progressId, isPointy) {

	    // Elements
	    var canvas = document.getElementById(canvasId);
	    var slider = document.getElementById(sliderId);
	    var progress = document.getElementById(progressId);
	    var context = canvas.getContext("2d");

	    // Event handler
	    slider.addEventListener("input", function () {
	    	update(slider.value);
	    }, false);

	    // Bounds
	    var width = canvas.width;
	    var height = canvas.height;

	    // Measure
	    var l = 0.60 * width;
	    var g = 0.15 * height;
	    var w = 0.05 * height;
	    var m = l / 2;

	    // Helpers
	    var centerY = height / 2;
	    var startX = (width - l) / 2;
	    var endX = startX + l;
	    var topY = centerY - g;
	    var bottomY = centerY + g;
	    var pointyFactor = isPointy ? (.3536 * w) : 0;

	    // Show UI
	    context.lineWidth = w;
	    update(0.0);

	    function update(prog) {
	    	progress.innerHTML = "Current progress: " + parseFloat(prog).toFixed(2);
	    	draw(context, prog);
	    }

	    function draw(canv, prog) {
	    	var dx = prog * m;
	    	var dy = prog * (m - g);
	    	var dg = prog * g;
	    	var dz = prog * pointyFactor;

	        // Reset canvas
	        canv.clearRect(0, 0, width, height);
	        canv.beginPath();

	        // Top line
	        canv.moveTo(startX + dx, topY - dy);
	        canv.lineTo(endX + dz, topY + dg + dz);

	        // Middle line
	        canv.moveTo(startX, centerY);
	        canv.lineTo(endX, centerY);

	        // Bottom line
	        canv.moveTo(startX + dx, bottomY + dy);
	        canv.lineTo(endX + dz, bottomY - dg - dz);

	        // Draw
	        canv.closePath();
	        canv.stroke();
	    }

	};

	DelightfulDrawable.newInstance("canvas", "slider", "prog", false);
	DelightfulDrawable.newInstance("canvas2", "slider2", "prog2", true);

</script>

[1]:http://www.google.com/design/spec/animation/delightful-details.html
[2]:http://en.wikipedia.org/wiki/Linear_interpolation