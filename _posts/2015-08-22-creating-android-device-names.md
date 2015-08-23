---
layout: post
title: "Creating AndroidDeviceNames"
description: "A story behind AndroidDeviceNames, a tiny Android library that transforms the device model name into something users can understand."
tags: [Android]
---

A few days ago, I open sourced [AndroidDeviceNames][0], a tiny Android library that transforms the device model into a user-friendly name. 

For example, `SM-N910W8` becomes `Samsung Galaxy Note 4` with a simple method call:

{% highlight java %}
DeviceNames.getDeviceName("SM-N910W8", "Unknown Device");
{% endhighlight %}

Currently, the library recognises about 400 devices. In case the device is not on the list, the above example returns `Unknown Device`. 

The idea for the library came while working on a side project where I need to show the device name to the user. Obviously, showing `SM-N910W8` is subpar. Searching for a solution, all I could find online was a somewhat obsolete [project][1] with a single `.properties` file containing `model = name` pairs. I guess the idea is to load the file into a [Properties][2] object.

I don't particularly like this. My side project manipulates bitmaps regularly and I'd like to avoid any unnecessary memory overhead. Since `Properties` extends `Hashtable`, overhead is expected. Here's the [hprof dump](https://gist.github.com/tslamic/df10d6c6fb7f654e02bc#file-devicenamesproperties-java):

![props_hprof_dump](http://i.imgur.com/tAqPrLO.png)

It finds the model name in `~O(1)` time but takes a whopping `59 kB` of memory to do it. In grand scheme of things, this is no problem for Android, but it can be improved. For instance, we could read the file, line by line and:

 1. discard the pair and continue, if it's not what we're looking for, or
 2. save it to a database, or
 3. remember the pair in a data structure with hopefully less memory overhead than `Properties`

Option 1. will only allocate a constant chunk of memory, i.e. the buffer size, but there's a high penalty to pay if we plan to search often and with different models, as we have to read the file over and over.

Option 2. is good, but messy. I'd have to be smart about database opening and closing, there's suddenly more code involved, I might clutter my API, etc. Hopefully I can avoid it.

Could `Map`, `ArrayMap` or `String[]` perform better than `Properties`? [Investigating](http://i.imgur.com/bEL3AAh.png) option 3. produces the following: 

| Data Structure | Retained Heap (kB) | Search time |
|---:|:---:|:---|
|[`Map`](https://gist.github.com/tslamic/df10d6c6fb7f654e02bc#file-devicenamesmap-java)| 62 | `~O(1)` |
|[`ArrayMap`](https://gist.github.com/tslamic/df10d6c6fb7f654e02bc#file-devicenamesarraymap-java)| 53 | `O(log)` |
|[`String[]`](https://gist.github.com/tslamic/df10d6c6fb7f654e02bc#file-devicenamesarray-java)| 51 | `O(n)` |
|[`String[], String[]`](https://gist.github.com/tslamic/df10d6c6fb7f654e02bc#file-devicenamestwoarrays-java)| 51 | `O(n)` |

Besides a nice insight in memory vs. time tradeoff, there's still quite a bit of memory involved. It does seem an `ArrayMap`  represents the middle ground between memory and performance, but [watch this][6] before jumping to conclusions.

Can things be improved if we don't need to be as flexible? For example, if the `model-name` pairs are embedded in a class as constants and reading a file is not required? Well, it makes [quite an impact](http://i.imgur.com/XC2olUn.png):

| Data Structure | Retained Heap (kB) | Search time | Factor
|---:|:---:|:---|:---|
|`Map`| 14 | `~O(1)` | 4
|`ArrayMap`| 5 | `O(log)` | 10
|`String[]`| 3 | `O(n)` | 17
|`String[]`,`String[]`| 3 | `O(n)` | 17

If our goal is to save memory, we should definitely avoid reading files. The factor column shows that, for example, `String[]` implementation with constants consumes 17x less memory than its file-reading counterpart.

So how do I effectively move the `model-name` pairs from a file to Java constants, because I'm sure as hell not writing them by hand?

It took me a few minutes to write a simple Python script that reads the `model-name` file and creates a populated Java data structure for me to copy and paste. Sure, more involved than plain Java, but straightforward.

Sacrificing a bit of flexibility for a memory efficient implementation, I could stop here. But would it be possible to eliminate memory overhead completely? This was more of an exploration question, born _not out of necessity_, but rather curiosity. 

I'd have to find a way to eliminate the data structure holding the `model-name` pairs. A naive but workable approach could be the following:

{% highlight java %}
if ("ASUS_Transformer_Pad_TF700T".equals(model)) {
    return "ASUS Transformer Pad TF700T";
} else if ("ASUS_T00J".equals(model)) {
    return "Asus ZenFone 5";
} else if ("ADR6350".equals(model)) {
    return "HTC Droid Incredible 2";
} // etc.
{% endhighlight %}

Ugly, but with no memory overhead and an _okay_ search performance of `O(n)`. 

I have to be careful though. When a `.class` file is loaded, it resides in the [method area][4] of a JVM. The method area is logically part of the heap, so it's futile trying to avoid overhead if the bytecode occupies the same amount of heap (or more) than a data structure would.

A neat way to look at bytecode is [`javap`](http://docs.oracle.com/javase/7/docs/technotes/tools/windows/javap.html). For example, on `Java(TM) SE Runtime Environment (build 1.7.0_79-b15)`, compiling

{% highlight java %}
public class Hello {
    public static void main(String[] args) {
        System.out.println("Hello World");
    }
}
{% endhighlight %}

then invoking `javap -c Hello` produces

{% highlight java %}
Compiled from "Hello.java"
public class Hello {
  public Hello();
    Code:
       0: aload_0       
       1: invokespecial #1 
       4: return        

  public static void main(java.lang.String[]);
    Code:
       0: getstatic     #2                 
       3: ldc           #3
       5: invokevirtual #4
       8: return        
}
{% endhighlight %}  

Each bytecode instruction consists of 1 byte, plus a fixed number of operand bytes. `-c` flag ensures the output includes byte offsets for each instruction. Therefore, the method size is roughly the same as the last byte offset. Above, `main` method size is 8 bytes, as this is the offset at the last instruction: `8: return`.

Extending the Python script, I now generate a simple Java method, `getDeviceName`, which includes the `if-else` comparisons. I need to ensure that:

 - the generated method size `mG` is smaller then the method size `mD` querying a data structure plus the total data structure memory, `sD`.
 - `mG` size is less than [65536 bytes][5] 

Benchmarking against the least consuming data structure, `String[]` gives

| File | Bytecode | Bytecode Size | Memory | Total
|----:|:--:|:--:|:--:|:--:|
|`String[]` | [bytecode](https://gist.github.com/tslamic/df10d6c6fb7f654e02bc#file-devicenamesarraynofile-bytecode) | 75 + 6233 | 3416 | 9724 |
|`Generated`| [bytecode](https://gist.github.com/tslamic/df10d6c6fb7f654e02bc#file-devicenames-bytecode)| 5691 | 0 | 5691

The generated file is a clear winner and passes the two requirements with flying colors. The only thing left to worry about is the search time. Being `O(n)`, it might prove to be unsatisfactory. 

A quick and simple optimization we can do right away is simple branching: taking the first model letter and compare it only against the models with the same starting letter. We're still stuck in `O(n)`, but for example, instead of 361 comparisons to get to `SPH-D600`, we now only do 61, which is about 6 times less.

Empirically, I'd like to see the worst case run below 16 milliseconds. And playing around with _Traceview_, the average running time is well below 5 milliseconds, which makes me at ease using the generated `.java` file in my side project.

[0]:https://github.com/tslamic/AndroidDeviceNames
[1]:https://github.com/meetup/android-device-names
[2]:http://docs.oracle.com/javase/7/docs/api/java/util/Properties.html
[4]:https://docs.oracle.com/javase/specs/jvms/se7/html/jvms-2.html#jvms-2.5
[5]:http://docs.oracle.com/javase/specs/jvms/se7/html/jvms-4.html#jvms-4.9.1
[6]:https://www.youtube.com/watch?v=ORgucLTtTDI