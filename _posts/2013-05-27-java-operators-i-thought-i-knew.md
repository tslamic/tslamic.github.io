---
layout:   post
title:    Java Operators I Thought I Knew
comments: true
---

Today, I stumbled upon [Java Tutorial Operators](http://docs.oracle.com/javase/tutorial/java/nutsandbolts/operators.html) page. I glanced the list, wondering if I know what each operator does:

<center>
  <figure>
    <img src="http://i.imgur.com/BuNAYbG.png" />
    <figcaption>Do you know them all?</figcaption>
  </figure>
</center>

As you might have guessed from the title, I failed. Luckily, I failed at only two operators. 

### The modulo operator %

While mostly defined in the same fashion, implementations of _modulo_ or _reminder_ operator differ when it comes to the sign of a result. For example, a modulo operator in Python might yield a different result than a modulo operator in Java, using the exact same data. Try it out for yourself - evaluate the expression `-1 % 2`, for instance.

Why?

It boils down to the decision language designers made when defining the operator. Recall that in an expression `a % b = c`,
`a` is said to be the _dividend_ and `b` the _divisor_. Some decided `c` should have the same sign as the dividend, others divisor. Java chose dividend, Python divisor.

Missing this fact could lead to some "fun" late night debugging sessions. Consider the following example:

```java
/*
 * Returns true if value is odd, false otherwise.
 */
static boolean isOdd(final int value) {
  return (value % 2) == 1;
}
```

It works wonderful with a positive argument. But since the definition of an odd number is quite natural for any negative integer too, does `isOdd(-1)` produce a correct result? 

Java uses the dividend when determining the sign of a modulo operation, so the argument we pass to `isOdd`. Therefore `-1 % 2 == -1`, and `isOdd(-1) == false`. The result is not what one would expect.

A quick fix could apply absolute value to a dividend. An even better solution, albeit a bit cryptic to an untrained eye, would be to just check the last bit of a number and avoiding the modulo operator altogether:

```java
/*
 * Returns true if value is odd, false otherwise.
 */
static boolean isOdd(final int value) {
  return (value & 1) == 1;
}
```

### The unsigned right shift >>>

The second operator I was having trouble with was `>>>`. Honestly, I couldn't tell the difference with `>>` operator, so I had to check the definition: `>>>`is an _unsigned_ right shift whereas `>>` is _signed_. If you're confused, read on. 

<center>
  <figure>
    <img src="http://upload.wikimedia.org/wikipedia/commons/thumb/7/76/Most_significant_bit.svg/300px-Most_significant_bit.svg.png" />
    <figcaption>The most significant bit is highlighted.</figcaption>
  </figure>
</center>

`>>` operator uses the most-significant bit value when shifting a bit pattern, whereas `>>>` _always_ uses `0`.

Suppose we're operating with 4-bit two's complement numbers. Take a binary number, e.g. `0001`. The most significant bit is `0`. Performing a shift, say `0001 >> 1`, the value you'll be shifting with is `0`.

Now consider the binary number `1001`. The most significant bit in this case is `1`, so the value you'll be shifting with is `1`.

<pre>
0001 >> 1 == 0000
1001 >> 1 == 1100
</pre>

Regardless of the most significant bit value, an unsigned right shift operator `>>>` always uses `0` as the shifting value. So `0001` will be shifted with `0`, as will `1001`:

<pre>
0001 >>> 1 = 0000
1001 >>> 1 = 0100
</pre>

You may notice there is no unsigned left shift operator. This is because the `<<`operator is already _unsigned_: `1000 << 1 == 0000`. In two's complement system, the most significant bit represents the sign. This simple example clearly shows that the sign is lost when performing the left shift. 

So why is there no _signed left shift_ operator? One could reason it's unnecessary: suppose `<<<` denotes it. Then, for example, `1001 <<< 1 == 1010`. The same result may be achieved using `>>`: `1001 >> 2 == 1010`.