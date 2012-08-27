---
layout: page
title: 2 Cent Tip - Dynamic rdesktop resolution.
date: 2010-06-28
comments: true
categories:
- 2 cent tip 
- Desktop Linux
- bash
---

So occasionally I do have to touch a Windows system, or use a Windows-only management tool (I'm looking at you VMware).  Not that I have any problem with Microsoft or Windows, I'm really just more comfortable in a Unix-like environment.  I do use the Open Source `rdesktop` utility to access Windows machine using version 5.0 of the Remote Desktop Protocol (RDP).

It's a handy utility, but I really wish it would give me an appropriate resolution based on the current resolution of my laptop's X Windows session.  There is, in fact, a command line flag to alter the geometry of the remote desktop window.  However, typing in `rdesktop -g 1280x1024` is much more tedious than typing in `rdesktop` on the command line interface.

<!-- more -->

So the simple solution is to put an alias in the `.bashrc` file, like so...

{% highlight bash %}
    # ~/.bashrc
    rdesktop='rdesktop -g 1280x1024'
{% endhighlight %}

However, on my laptop, the max resolution without an external monitor is 1600x900.  So I still have to override the setting with `rdesktop -g 1280x1024` on the command line, any time I am running without an external monitor.

Another solution would be to use `awk` to find a smaller resolution from `xrandr`, then set that resolution in my `rdesktop` alias.

{% highlight bash %}
    # ~/.bashrc
    RDESKTOP_SIZE=`xrandr | awk '{getline; getline; getline; print $1; exit;}'`

    alias rdesktop='rdesktop -g $RDESKTOP_SIZE'
{% endhighlight %}

This `awk` command will skip the first three lines, from the xrandr output.  That is two header lines, and the maximum resolution on the following line, which we'll skip.  I'll take the next highest resolution and set that in the variable `$RDESKTOP_SIZE`.  Finally, I'll use the variable `$RDESKTOP_SIZE` with the `-g` switch in `rdesktop` to properly set my alias.

Let us say you have two big monitors plugged into a port replicator.  Perhaps you only want the `rdesktop` window to cover part of one screen.  The following code will keep my `rdesktop` window smaller than one of the 1920x1200 screens on a spanned 3840x1200 resolution.

{% highlight bash %}
    # ~/.bashrc
    RDESKTOP_SIZE=`xrandr | awk '{getline; getline; getline; print $1; exit;}'`

    if [ $RDESKTOP_SIZE == "3840x1200" ]; then
        RDESKTOP_SIZE="1820x1100"
    fi

    alias rdesktop='rdesktop -g $RDESKTOP_SIZE'
{% endhighlight %}

In other words, if I have two monitors at 1920x1200 each, their combined resolution is reported by `xrandr` as 3840x1200.  I want to override the detected resolution to 100 pixels less than the resolution on one screen.  Therefore I set the `$RDESKTOP_SIZE` variable to 1820x1100, before setting my `rdesktop` alias in `.bashrc`.
