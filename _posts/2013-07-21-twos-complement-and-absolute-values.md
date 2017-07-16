---
layout: post
title: "Two's Complement And Absolute Values"
subtitle: "Two's Complement and a quirky property that comes with it."
author: "Tadej"
comments: true
---

<script type="text/javascript"
 src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML">
</script>

If I ask you what the result of `Math.abs(-10)` is, would you guess it's `10`? How about `Math.abs(-2147483648)`? If you think the answer is `2147483648`, you're wrong.

### How it all starts

A bit can hold only one of two values, say `0` and `1`. An 8-bit value, for example, is
a combination of 8 bits _glued together_:
<pre>
001: [0] [0] [0] [0] [0] [0] [0] [0]
002: [0] [0] [0] [0] [0] [0] [0] [1]
003: [0] [0] [0] [0] [0] [0] [1] [0]
004: [0] [0] [0] [0] [0] [0] [1] [1]
...
256: [1] [1] [1] [1] [1] [1] [1] [1]
</pre>

There are 256 possible values. Being presented with the table above and no additional information, a natural question arises: <em>what do this combinations of bits represent?</em>

Knowing computers operate with binary numbers, a good guess would be this are just binary representations of numbers. Lo and behold, you've discovered an _unsigned_ 8-bit integer representation system. Converting to decimal, the values yield:

<pre>
001: [0] [0] [0] [0] [0] [0] [0] [0] = 0
002: [0] [0] [0] [0] [0] [0] [0] [1] = 1
003: [0] [0] [0] [0] [0] [0] [1] [0] = 2
004: [0] [0] [0] [0] [0] [0] [1] [1] = 3
...
256: [1] [1] [1] [1] [1] [1] [1] [1] = 255
</pre>

Notice what _unsigned_ means - the values we've calculated are all missing a sign, so we naturally assume they're non-negative. Also, we're only able to represent integers from 0 to 255.

To add negative numbers and make our system _signed_, let's make a wild guess and choose the first bit to represent the sign: `0` means the number will be positive, `1` negative:
<pre>
[0] [0] [0] [0] [0] [0] [0] [0] = +0
[0] [0] [0] [0] [0] [0] [0] [1] = +1
[0] [0] [0] [0] [0] [0] [1] [0] = +2
...
[0] [1] [1] [1] [1] [1] [1] [1] = +127
</pre>
<pre>
[1] [0] [0] [0] [0] [0] [0] [0] = -0
[1] [0] [0] [0] [0] [0] [0] [1] = -1
[1] [0] [0] [0] [0] [0] [1] [0] = -2
...
[1] [1] [1] [1] [1] [1] [1] [1] = -127
</pre>

Voila! Our very first 8-bit signed integer system. Being very intuitive, this is a well known system called _signed magnitude_, used by some ancient computers, such as IBM 7090.

<center>
  <figure>
    <img src="http://ed-thelen.org/comp-hist/BRL61-0546.jpg"/>
    <figcaption>IBM 7090</figcaption>
  </figure>
</center>

### Enter Two's Complement

Through time, people had come up with clever ways of storing integers, so that common math problems are simple to implement, efficient, and have no duplicate values.

<blockquote>
For an arbitrary n-bit binary number, to get its opposite representation, first invert the number, then add 1.
</blockquote>

This simple algorithm forms Two's Complement, the most commonly used signed number representation system in use today.

Let's do an example: find the opposite value of the binary number `00001010`:
<pre>
inverted(00001010) = 11110101
    add1(11110101) = 11110110
</pre>

Let's check what the opposite number of `11110110` is - if the algorithm is well-defined, the result should be `00001010`:
<pre>
inverted(11110110) = 00001001
    add1(00001001) = 00001010
</pre>

Now for a more juicy example. What's the opposite binary number of `00000000`?
<pre>
inverted(00000000) =  11111111
    add1(11111111) = 100000000
</pre>

The result is a 9-bit number in an 8-bit number system. This is called an _overflow_ since the value is, well, overflowing the 8-bit size restriction. The overflow is usually discarded, leaving us with the first eight bits from the right, `00000000`. We just showed that there is exactly one representation of `0`.

### Sign

In Two's Complement, the most significant bit represents the sign. `0` means the number will be positive, `1` negative. This means that with _n_ bits, you can only use _(n-1)_ bits to represent the number, as one bit is reserved to denote the sign.

You may think of, e.g., `01001010` as a compound value consisting of `0` (sign) and `1001010` (actual binary number).

### But I can't think in binary

Converting a binary value `01001010` to decimal is simple. Ignoring the most significant bit, we get

$$ 0 \times 2^{0} + 1 \times 2^{1} + 0 \times 2^{2} + 1 \times 2^{3} + 1 \times 2^{6} = 138 $$

What about `10001010`?

This is a negative value, since the most significant bit is `1`. To get the actual value, we can apply Two's Complement algorithm, ignoring the first bit:

<pre>
inverted(0001010) = 1110101
    add1(1110101) = 1110110
</pre>

Then,

$$ 0 \times 2^{0} + 1 \times 2^{1} + 1 \times 2^{3} + 0 \times 2^{4} + 1 \times 2^{5} + 1 \times 2^{6} + 1 \times 2^{7} = 118 $$

Because the most significant bit is `1`, the decimal number we calculated is negative. Therefore, `10001010 == -118`.

### What about the bounds?

In 8-bit unsigned integer system, we could represent integers from 0 to 255. With 8-bit sign and magnitude, we could represent numbers from -127 to 127. What about Two's Complement bounds?

Let's first look at 3-bit two's complement binary numbers and its decimal values:

<center>
<table style="width:300px; text-align:center">
  <tr>
    <th>Bits</th>
    <th>Values</th>
  </tr>

  <tr>
    <td>000</td>
    <td>0</td>
  </tr>

  <tr>
    <td>001</td>
    <td>1</td>
  </tr>

  <tr>
    <td>010</td>
    <td>2</td>
  </tr>

  <tr style="border-bottom: 2px solid #ccc;">
    <td>011</td>
    <td>3</td>
  </tr>

 <tr>
    <td>100</td>
    <td>-4</td>
  </tr>

  <tr>
    <td>101</td>
    <td>-3</td>
  </tr>

  <tr>
    <td>110</td>
    <td>-2</td>
  </tr>

  <tr>
    <td>111</td>
    <td>-1</td>
  </tr>

</table>
</center>

Since one bit represents the sign, there are only two bits available for binary numbers. The upper bound seems to be \\(3 = (2^{2} - 1) \\). The lower bound is \\( -4 = -2^{2} \\)

Let's do the same thing for 4-bit numbers:

<center>
<table style="width:300px; text-align:center">
  <tr>
    <th>Bits</th>
    <th>Values</th>
  </tr>

  <tr>
    <td>0000</td>
    <td>0</td>
  </tr>

  <tr>
    <td>0001</td>
    <td>1</td>
  </tr>

  <tr>
    <td>0010</td>
    <td>2</td>
  </tr>

  <tr>
    <td>0011</td>
    <td>3</td>
  </tr>

 <tr>
    <td>0100</td>
    <td>4</td>
  </tr>

  <tr>
    <td>0101</td>
    <td>5</td>
  </tr>

  <tr>
    <td>0110</td>
    <td>6</td>
  </tr>

  <tr style="border-bottom: 2px solid #ccc;">
    <td>0111</td>
    <td>7</td>
  </tr>

 <tr>
    <td>1000</td>
    <td>-8</td>
  </tr>

  <tr>
    <td>1001</td>
    <td>-7</td>
  </tr>

  <tr>
    <td>1010</td>
    <td>-6</td>
  </tr>

  <tr>
    <td>1011</td>
    <td>-5</td>
  </tr>

 <tr>
    <td>1100</td>
    <td>-4</td>
  </tr>

  <tr>
    <td>1101</td>
    <td>-3</td>
  </tr>

  <tr>
    <td>1110</td>
    <td>-2</td>
  </tr>

  <tr>
    <td>1111</td>
    <td>-1</td>
  </tr>

</table>
</center>

Again, one bit represents the sign, so there are three bits available for binary numbers. The upper bound seems to be \\( 7 = 2 ^ {3} - 1 \\). The lower bound is \\( -8 = -2^{3} \\)

Notice the bounds pattern for number of bits:

<center>
<table style="width:550px; text-align:center">
  <tr>
    <th>Number of bits</th>
    <th>Lower bound</th>
    <th>Upper bound</th>
  </tr>

  <tr>
    <td>3</td>
    <td>-4</td>
    <td>3</td>
  </tr>

  <tr>
    <td>4</td>
    <td>-8</td>
    <td>7</td>
  </tr>

  <tr>
    <td>5</td>
    <td>-16</td>
    <td>15</td>
  </tr>

  <tr>
    <td>...</td>
    <td>...</td>
    <td>...</td>
  </tr>

 <tr>
    <td>n</td>
    <td>\( -2^{n - 1} \)</td>
    <td>\( 2^{n - 1} - 1 \)</td>
  </tr>

</table>
</center>

### The grand finale

Let's look at one more example. What is the opposite value of binary number `100`?

<pre>
inverted(100) = 011
    add1(011) = 100
</pre>

The opposite value of `100` is the exact same value. Weird. Let's do one more. What's the opposite value of `1000`?

<pre>
inverted(1000) = 0111
    add1(0111) = 1000
</pre>

Applying inductive reasoning, this holds for any _n-bit_ number. But what exactly are this values?

<center>
<table style="width:500px; text-align:center">
  <tr>
    <th>Number of bits</th>
    <th>Bit value</th>
    <th>Decimal value</th>
  </tr>

  <tr>
    <td>3</td>
    <td>100</td>
    <td>-4</td>
  </tr>

  <tr>
    <td>4</td>
    <td>1000</td>
    <td>-8</td>
  </tr>

  <tr>
    <td>5</td>
    <td>10000</td>
    <td>-16</td>
  </tr>

   <tr>
    <td>...</td>
    <td>...</td>
    <td>...</td>
  </tr>

  <tr>
   <td>n</td>
   <td></td>
   <td>\(-2^{n - 1}\)</td>
  </tr>

</table>
</center>


The values are lower bounds. We've shown that in Two's Complement, the opposite value of a lower bound is still the lower bound.

This is relevant because `-2147483648 == Integer.MIN_VALUE`, a lower bound for `int` type in Java. And as such, `Math.abs(-2147483648) == -2147483648`.
