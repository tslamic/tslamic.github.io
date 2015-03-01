---
layout: post
title: "On Advanced Task Killer"
description: "How does its internals tick?"
tags: [android, task killer, memory]
---

There’s not a single Android enthusiast that doesn’t know the Advanced Task Killer app.

It’s supposedly the #1 task management app on Google Play, designed to kill applications and boost memory. Fiddling with system management is generally a bad idea, so I was quite curious what kind of black magic are they doing under the hood. With all the popularity and more than half a million downloads, they must be doing something right. Right?

Firstly, an important question. Why would anyone want to _kill_ an app? I agree with [Reto Meier](http://blog.radioactiveyak.com/2010/05/when-to-include-exit-button-in-android.html):

> … in my experience what users really want is an unambiguous way to guarantee that an app will stop consuming resources (battery, CPU cycles, data transfer, etc). Many perceive that an exit button implements this requirement and ask for it to be added. Developers, looking to please their users, obligingly add one…

With seamless state restoration, Android (and iOS) apps seem to run forever. They give no real _exit_ notion. Misunderstanding this non-desktop behaviour is, I’m guessing, why apps such as Advanced Task Killer (ATK henceforth) are well received. Few realize Android is clever enough to manage itself much better than we ever will, and there’s no need to interfere. In development world, manual or forced quitting is actually [frowned upon](http://stackoverflow.com/a/2034238/905349), or, as Romain Guy [explains:](https://groups.google.com/d/msg/android-developers/G_D3pKnGLt0/0mFuZnjxjP4J) 

>… the system handles it automatically. That's what the Activity lifecycle (especially onPause/onStop/onDestroy) is for. No matter what you do, do not put a "quit" or "exit" application button. It is useless with Android's application model.

In reality, quiting could make matters worse. Killed apps might be restarted if they are, for example, receiving system events. Same goes for some services. You esentially force Android to do more work and consume more resources, such as battery life, to achieve absolutely nothing.

With this in mind, it’s fair to speculate ATK is redundant. To see it tick, the initial step is to observe its source. Reversing an apk is, with tools available online, a piece of cake. Given the app feel, I didn’t expect it to be obfuscated. To my surprise, it was. Browsing through, I had to admit defeat - no `killProcess` invocation anywhere, which I was expecting. At least not explicitly, that is.

For fun (and no profit), let's try to recreate the app. 

First, we need to get a list of all activities currently running. ATK can also extract running services. We're going to ignore them, but you can try and add them yourself - it's easy.

{% highlight java %}
/*
 * Returns the list of all available launcher Activities.
 */
private List<ResolveInfo> getAvailableActivityLaunchers() {
    final PackageManager manager = getPackageManager();
    final Intent intent = new Intent(Intent.ACTION_MAIN, null);
    intent.addCategory(Intent.CATEGORY_LAUNCHER);
    return manager.queryIntentActivities(intent, 0);
}
{% endhighlight %}

Next, we want to filter out the activities currently running, but ignore the ones associated with the system:

{% highlight java %}
/*
 * Create a mapping from a process name to its ResolveInfo.
 */
private Map<String, ResolveInfo> getAvailableActivityLaunchersMap() {
    final List<ResolveInfo> list = getAvailableActivityLaunchers();
    final HashMap<String, ResolveInfo> map = new HashMap<String, ResolveInfo>(list.size());
    for (ResolveInfo info: list) {
        map.put(info.activityInfo.processName, info);
    }
    return map;
}

/*
 * Return a filtered list of currently running Activities.
 */
private List<ResolveInfo> getAvailableLaunchersCurrentlyRunning() {
    final ActivityManager manager = getActivityManager(this);
    final Map<String, ResolveInfo> map = getAvailableActivityLaunchersMap();

    final List<ActivityManager.RunningAppProcessInfo> runningApps = manager.getRunningAppProcesses();
    final List<ResolveInfo> available = new ArrayList<ResolveInfo>(runningApps.size());
    for (ActivityManager.RunningAppProcessInfo runningApp: runningApps) {
        if (!skip(runningApp)) { // skips system-related Activities
            final ResolveInfo info = map.get(runningApp.processName);
            if (null != info) {
                available.add(info);
            }
        }
    }
    return available;
}
{% endhighlight %}

This is, believe it or not, the gist of ATK. The only thing missing is the ability to _kill_ a selected list item. ATK does it by invoking `activityManager.restartPackage(packageName)`. Initially, [this method](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/2.0_r1/android/app/ActivityManager.java#ActivityManager.restartPackage%28java.lang.String%29) had the system perform a force stop of everything associated with the given application package (killed processes with same uid, stopped running services, removed activities, etc). Heavily abused by task killers, it caused all sorts of issues. Android engineers eventually had enough. They've changed its behaviour, leaving [this comment](https://android.googlesource.com/platform/frameworks/base.git/+/03abb8179f0d912e6dabfc0e2b0f129d85066d17): 

> Kill the task killers. <br/><br/> The `ActivityManager.restartPackage()` API is now deprecated, and no longer
allows applications to mess up the state of other applications.  This was
being abused by task killers, causing users to think their other applications
had bugs. <br/><br/> A new API is introduced for task killers,
`ActivityManager.killBackgroundProcesses()`, which allows these applications
to kill processes but only the same amount that the out of memory
killer does, thus causing no permanent damage.  The old `restartPackage()`
API is now a wrapper for calling this new API.

From API level 8, calling `restartPackage` is therefore the same as calling `killBackgroundProcesses`. Following the source code from, let's say, `ActivityThread.handleLowMemory()` which is presumably called when memory becomes tight, we end up in the same method: `killPackageProcessesLocked`. It's safe to assume ATK provides the exact same functionality as the system. The only difference is that ATK explicitly selects a package to be killed, whereas the system pick is based on an algorithm. Since the system knows its internals, it's much better at determining when and what to do when resource management is required. 

With this in mind, one could argue ATK is harmless. The real problem is us, killing apps at random, without knowing what they do behind the scenes. We should stop interfeering and our system will run just fine.

To finish off the fake ATK, we: 

- show the `getAvailableLaunchersCurrentlyRunning()` list in a `ListView`
- add a listener that, on item click, triggers the `killBackgroundProcesses()` method.

You can view the complete implementation, with very appealing UI, on [Github](https://github.com/tslamic/AndroidExamples/tree/master/TaskKiller/TaskKiller). In the near future, I plan to write a post about the _out-of-memory killer_. Some of the topics I'll touch are already included in the app, so just ignore them for now.
