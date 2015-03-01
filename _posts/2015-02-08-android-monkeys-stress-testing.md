---
layout: post
title: "Android, monkeys and stress testing"
description: "Automating the monkey and heap dumps."
tags: [android, monkey, memory, automation, dumpey]
---

As end users, we’re terrible. 

We’ll microwave a fork, pull doors with a big red sign saying push and sue a company for serving us a hot cup of coffee. We’ll grab a supposedly waterproof phone and toss it in a full tub. Then we’ll go online and laugh at the schmuck trying to drill through a wall he’s standing on. 

<center>
  <figure>
    <img src="http://i.imgur.com/X17puIB.gif"/>
    <figcaption>Can you imagine designing a product that can handle such superusers? </figcaption>
  </figure>
</center>

Creating software with good user experience is hard. Relying on intuition is naive, but expecting a product to be used in a predictable (or, y’know, logical) manner is just wrong. 

_Stress testing_ puts great emphasis on availability and error handling, under a load few consider normal. It indicates just how robust our product really is.

Android - or, more precisely, its debug bridge -  has a very handy tool called _the monkey_. It’s purpose is to stress-test your app by generating pseudo-random user events in a repeatable manner. It’s as if a monkey was ordered to test your app. It’ll tap, swipe, click and drag no matter what’s on the screen. 

A typical command to start the monkey is:

{% highlight bash %}
$ adb shell monkey -p your.package.name 5000
{% endhighlight %}

where `-p` denotes your package name. `5000` is the amount of events generated. 

Another useful option is the seed parameter. If you re-run the monkey with the same seed value, it will generate the exact same sequence of events. Just add the `-s <number>` option to your command: 

{% highlight bash %}
$ adb shell monkey -p your.package.name -s 12345 5000
{% endhighlight %}

This is handy when you encounter crashes while using the monkey. Apply the fix, re-run the monkey with the same seed value and validate.

Sometimes, you’ll have multiple devices to run the monkey on. Suppose `adb devices` yields

<pre>
List of devices attached 
192.168.56.102:5555		device
0749e0c421157449		device
017961c7d19f9615		device
</pre>

Executing any adb shell command results in `error: more than one device and emulator`. You need to explicitly tell the adb which device or emulator one to use, by using `-s <serialNumber>` option:

{% highlight bash %}
$ adb -s 192.168.56.102:5555 shell monkey -p your.package.name -s 12345 5000
{% endhighlight %}

This will run the monkey on a device or emulator with a serial number `192.168.56.102:5555`. To run the monkey on all attached emulators and devices, you have to execute a separate command for each one. 

Numerous other options are [available](http://developer.android.com/tools/help/monkey.html) to fine-tune the monkey. Perhaps `--hprof` is an option worth mentioning. If set, it dumps the heap immediately before and after running. This can be quite useful for memory profiling. 

However, according to this [old post](http://stackoverflow.com/a/8433740/905349), the `--hprof` option seems to be ignored. Moreover, the dumps are put in `data/misc` folder. It may be impossible to extract them.

To tackle the two problems, running the monkey on multiple devices and making heap dumps, I've created Dumpey, a collection of (currently) three UNIX scripts:

 - `monkey` capable of running the monkey on all attached devices/emulators
 - `memdmp` capable of extracting and converting heap dumps from a device/emulator to a local drive
 - `dumpey` capable of extracting and converting heap dumps before and after running the monkey, on all attached devices/emulators

Using Dumpey is easy. For example

{% highlight bash %}
$ ./monkey -p your.package.name -s 12345 -e 5000
{% endhighlight %}

runs the monkey on all attached devices with a seed number `12345`, generating `5000` random events. 

To avoid using DDMS to do a heap dump, use `memdmp`:

{% highlight bash %}
$ ./memdmp -s SH48HWM03500 -p your.package.name -f heapdumps/my_heap_dump.hprof
{% endhighlight %}

This will extract a converted memory heap dump to `heapdumps/my_heap_dump.hprof` file. All you have to do is open it with MAT.

Combine memory dumps and monkey with `dumpey`:

{% highlight bash %}
$ ./dumpey -p your.package.name -bad heapdumps/
{% endhighlight %}

This extracts converted memory heap dumps, runs the monkey, and extracts converted memory heap dumps again. It does this on all attached devices or emulators. If, for example, you only have one device attached, `SH48HWM03500`, then running `dumpey` with the above command will generate two files in `heapdumps` folder: 

- `SH48HWM03500-before.hprof`
- `SH48HWM03500-after.hprof`

As the names suggest, they are converted memory heap dumps from a specific device done before and after running the monkey. Again, all you have to do is open them in MAT. 

To read more about Dumpey, visit the [Github repo](https://github.com/tslamic/Dumpey). Happy stress testing!