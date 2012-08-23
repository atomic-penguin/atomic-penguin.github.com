---
layout: page
title: 2 cent Linux tip using find and tar for a selective backup 
date: 2011-07-11
comments: true
categories:
- 2 cent tip
- wine 
- backup
---

So I\'m blowing away, and re-installing, my Steam Bottle on [Codeweaver\'s](http://codeweavers.com) Crossover games.  I was hoping to get some in-game overlay support for the Community/Friends features of Steam.  I really really want to hang on to Team Fortress/Left 4 Dead settings though.

Using the `find` command we can solve this problem fairly easy.

So the basic requirements in this scenario boil down to finding all the config files under the `~/.cxgames` tree. I have an `~/etc` directory to keep a backup of important things such as settings and handy scripts. I will copy my tar file to the `~/etc` directory so I can find it easily later. However, I don't want every `*.cfg` file under my Steam bottle, just the ones for those specific games (Team Fortress and Left 4 Dead). When I run the following command, I\'ll discover those games have a common top-level directory, *~/.cxgames/Steam/drive_c/Program Files/Steam/steamapps*.

{% highlight bash %}
    find ~/.cxgames -name "*.cfg"
{% endhighlight %}

When I run the following command, however, I'll get an error due to spaces in the directory/filename structure.  Note the backslash in "Program Files" is an escaped space for the shell to properly interpret this.

{% highlight bash %}
    find .cxgames/Steam/drive_c/Program\ Files/Steam/steamapps/ | xargs tar -rf ~/etc/steam-settings.tar
{% endhighlight %}

The find command has a switch `-print0` to deal with this, and `xargs` will process this find output format with the `-0` flag.  The `-print0` and `-0` flags tell these system utilities to print or use null-terminated strings, making it easier for shell utilities to consume the spaces in the filenames.  Therefore the following command will get the files I need:

{% highlight bash %}
    find ~/.cxgames/Steam/drive_c/Program Files/Steam/steamapps/ -name "*.cfg" -print0 | xargs -0 tar -rf ~/etc/steam-settings.tar
{% endhighlight %}

Restoring is as simple as running the following command from within my home directory.

{% highlight bash %}
    tar xvf ~/etc/steam-settings.tar
{% endhighlight %}

Perhaps some day we'll have a Linux steam client, and a promotional TF2 item named the **Unix Pipe**.
