---
layout: post
title: "Disabling VerticalSync in KDE for Steam4Linux Beta"
date: 2013-01-15 10:16
comments: true
categories: gaming linux steam kde kwin vsync VerticalSync v-sync
---

According to [Wikipedia](http://en.wikipedia.org/wiki/Screen_tearing#V-sync), 
Vertical synchronization (or VerticalSync) is

`an option in most systems, wherein the video card is prevented from doing anything visible to the 
display memory until after the monitor finishes its current refresh cycle.`  

In layman's terms, when Vertical synchronization is turned on, many aspects
of video processing are locked to the reported refresh rate of your display
hardware.  In other words, if your fancy LCD screen reports 60 Hz. is its own
supported refresh rate, then games or video playback will be capped at 60 frames
per second when Vertical synchronization is turned on.

<!-- more -->

I have seen a couple reports that turning off [VSync](http://www.reddit.com/r/tf2/comments/15o98u/psalinux_vsync_enabled_by_default_even_if_it_says/)
doesn't actually disable VSync in Linux Steam games like Team Fortress 2.  The proposed workaround in this reddit thread was toggling
the option on, and off, until VSync was actually turned off.  Searching through the [steam-for-linux issue tracker](https://github.com/ValveSoftware/steam-for-linux)
last night, I found this [solution](https://github.com/ValveSoftware/steam-for-linux/issues/271) on an unrelated AMD TearFree bug report.

The solution offered by Github user dedoz, worked perfectly for me.  It turns out this is not a bug within any game, or Linux Steam, 
but rather a default option in compositing window managers such as KDE.  By default KDE turns on VSync for OpenGL rendering.  By 
changing the following setting, I am now able to toggle the option `mat_vsync 1` to enable VSync, or `mat_vsync 0` to disable VSync 
in-game from a game configuration file.  I have no idea, how to go about changing this in other window manager/desktop environments
suffering from the same problem.  If you encounter such a problem on another desktop environment, or find another workaround, be sure to leave a comment.

1. First locate kwinrc.  Mine lives in `~/.kde/share/config/kwinrc`

```bash
$ locate kwinrc
$HOME/.kde/share/config/kwinrc
/usr/share/kubuntu-default-settings/kde4-profile/default/share/config/kwinrc
/usr/share/kubuntu-netbook-default-settings/share/config/kwinrc
```

2. Next, find the `[Compositing]` section in `kwinrc`.  Add the line `GLVSync=false` to this section.

```bash
$ vim $HOME/.kde/share/config/kwinrc

[Compositing]
GLVSync=false
OpenGLIsUnsafe=false

<esc>:wq
```

3. Finally, use `qdbus` to call `org.kde.kwin.reconfigure()`.

```bash
$ qdbus org.kde.kwin /KWin reconfigure
```

Whether you want to play with Vertical synchronization on, or off, while in-game will come down to personal preference.
The window manager probably should not enforce Vertical synchronization being on by default, while in-game however.  Ideally, 
the in-game setting should be deferred to by the window manager.  The agnostic approach here, is to turn off enforcement
within the window manager, so that the user can decide on the setting which they prefer in-game.
