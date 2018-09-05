---
layout: post
title: "Simple Java Puzzler"
subtitle: "Comparing Integer objects can be tricky."
author: "Tadej"
comments: true
---

My first ever blog post was about [Java operators I thought I knew](http://tadej.me/java-operators-i-thought-i-knew/). Ironically, after working with Java daily for the last couple of years, I still find things I had no clue about.

My friend Massimo asked this the other day. Given

```java
Integer a = 100;
Integer b = 100;
Integer c = 200;
Integer d = 200;

boolean ab = (a == b);
boolean cd = (c == d);

System.out.println(ab + ", " + cd);
```

what is printed to stdout? 

Using `Integer` instead of an `int`, each variable points to an object. `==` checks for object equality and will compare memory locations. As we're not reassigning any variables, each references a different object and thus the output should be `false, false`, no?

Wrong.

The reasoning above is not wrong but misses the most important piece of this puzzle. When Java encounters code like `Integer a = 100;` [the compiler converts it](https://docs.oracle.com/javase/tutorial/java/data/autoboxing.html) into `Integer a = Integer.valueOf(100);` and since we all know what `Integer.valueOf` does, we never bother to read the docs.

Here's the [SE7 Javadoc](https://docs.oracle.com/javase/7/docs/api/java/lang/Integer.html#valueOf(int)):

> Returns an Integer instance representing the specified int value. If a new Integer instance is not required, this method should generally be used in preference to the constructor Integer(int), as this method is likely to yield significantly better space and time performance by caching frequently requested values. This method will always cache values in the range -128 to 127, inclusive, and may cache other values outside of this range.

The last sentence spills the beans. `Integer.valueOf` will always cache values in the interval `[-128,127]`. Here's the implementation:

```java
public static Integer valueOf(int i) {
  assert IntegerCache.high >= 127;
  if (i >= IntegerCache.low && i <= IntegerCache.high)
    return IntegerCache.cache[i + (-IntegerCache.low)];
  return new Integer(i);
}
```

[`IntegerCache`](http://grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/7u40-b43/java/lang/Integer.java#Integer.IntegerCache) caches all `Integer` objects between -128 and the upper bound, which will be at least 127, but can be extended by using `AutoBoxCacheMax` VM property. 

Returning to the puzzle, we now know

```java
Integer a = 100;
Integer b = 100;
boolean ab = (a == b);
```

both `a` and `b` will point to the same object as non-primitive 100 is cached, meaning `==` will yield `true`. 

Similarly, assuming `AutoBoxCacheMax` remains at default 127, 

```java
Integer c = 200;
Integer d = 200;
boolean cd = (c == d);
```

`c` and `d` will not be cached, as 200 is out of bounds and therefore a new `Integer` object will be assigned to each variable. `==` will then compare two distinct memory locations and yield `false`.

Putting the two together, we can now confidently say the output is `true, false`, which is - unsurprisingly now - the correct answer. 
