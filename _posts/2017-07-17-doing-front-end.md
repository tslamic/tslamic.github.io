---
layout: post
title: "On Doing Front-End Stuff"
subtitle: "And how I should probably avoid it."
author: "Tadej"
comments: true
---

Every now and then I need a reality check on how useless I am when it comes to front-end development.

After deciding to revamp my blog and spending ridiculous amount of time picking a new theme, I always feel obliged to add some personal touches to it. More often than not, this is a bad idea. After hours of fiddling with CSS and exhausting all the curse words in my vocabulary, I tell myself over and over again this has been the last time, while reverting back to square one.

This time, I tried to load big images asynchronously, and fade them in once loaded.

My new blog theme, [BlackrockDigital Clean Blog](https://github.com/BlackrockDigital/startbootstrap-clean-blog-jekyll), contains a header with a relatively large background image (see above), and it sometimes takes a while to load. Here's a simplified HTML structure imposed by the theme:

```html
<body>
  <header class="header" style="background-image: url('url')">
    <div class="container">
      <p> Navigation stuff here</p>
    </div>
  </header>
  <div class="container">
    <p> Content </p>
  </div>
</body>
```

Remember, I'm pretty bad when it comes to front-end. So the less I change, the better. But as it stands, I feel that `background-image` it's not really set out to be animated.

My plan is to remove the style attribute, introduce a new `img` tag inside the header representing background,
then load the image in Javascript and update its `src`.

Here's what I did:

```html
<body>
  <header class="header">
    <img class="header-bg"
         src="data:image/gif;base64,R0lGODlhAQABAAD/ACwAAAAAAQABAAACADs=" />
    <div class="container">
      <p> Navigation stuff here</p>
    </div>
  </header>
  <div class="container">
    <p> Content </p>
  </div>
</body>
```
I wrongly assumed it's straightforward to position the `img` behind the header using something like:

```css
.header-bg {
  position: absolute;
  width: inherit;
  height: inherit;
  z-index: -1;
}
```

with Bootstrap adding another class to the img tag, ensuring it's responsive:

```css
.img-responsive {
  display: block;
  max-width: 100%;
  height: auto;
}
```

I spent _a lot_ of time trying to make the above work, adding and removing various CSS elements. Epic fail.

I decided to try a more brute solution with Javascript. Reverting to the original HTML structure,
bar the style attribute:

```html
<body>
  <header class="header">
    <div class="container">
      <p> Navigation stuff here</p>
    </div>
  </header>
  <div class="container">
    <p> Content </p>
  </div>
</body>
```

I came up with the following function:

```javascript
function loadBackground(url, animDuration) {
  $('.header-bg').remove(); // Safety net.

  $('<img />', {
    'src': url,
    'class': 'header-bg',
    'css': {
      'display': 'none'
    }
  }).load(function() {
    var header = $(".header");
    var img = $(this);

    img.css({
      "position": "absolute",
      "z-index": -1,
      "width": header.width(),
      "height": header.height(),
    });

    header.prepend(img);
    img.attr("src", img.src).fadeIn(animDuration);
  });
}
```

It worked - but it looked pretty mediocre, and I have no clue what form factors
and responsiveness I broke introducing it. In the end, adding +1 to the number
of times I've gone down the front-end road and failed, I reverted back to the original, resized and optimized the images a bit, and called it a day.

Here's the [jsFiddle](https://jsfiddle.net/8ut4gsup/) in case you're interested.
