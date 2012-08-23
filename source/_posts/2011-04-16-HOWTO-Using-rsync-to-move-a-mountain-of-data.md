---
layout: page
title: HOWTO Using rsync to move a mountain of data
date: 2011-04-16
comments: true
categories:
- HOWTO
- rsync
- storage
---

In this installment of my blog, I want to document the proper use of rsync for folks who are tasked with moving a large amount of data.  I\'ll even show you a few things you can do from the command line interface to extend the built-in capability of rsync using a little bash-scripting trickery.

I use rsync to migrate Oracle databases between servers at least a few times per year.  In a snap, its one of the easiest ways to clone a database from a Production server to a Pre-Production/Development server or even a Virtual Machine.  You don't have to have a fancy Fibre-Channel or iSCSI storage array attached to both servers, in order to do a data LUN clone, thanks to rsync.

I hope you enjoy this in-depth article.  Please feel free to comment: if you need clarification, find it useful, or something I wrote is just plain wrong.

## The Tools

Three things you'll need for this exercise:

1. `screen` - lets you detach a console session, and can be re-attached after logging out and walking away.
2. `rsync` - the swiss-army knife of copy/archiving programs.
3. `data` - any will do. Whether its a large group of /home directories, or even a live Oracle database.

## Basics of screen

1. Simply type `screen` in the command-line interface.
2. Kick off a large rsync job that may take several hours to finish
3. `ctrl-a d` detaches your screen session.  You may then proceed to logout, walk away, and go grab a lunch, or two.
4. Log back in to the server later, and re-attach your session by running `screen -r`.
  * If your connection to the server was interrupted by an unreliable network, screen may throw an error saying the session is already attached, however screen -dr will de-tach and re-attach the session with no problem.

## Basics of rsync

There is a daunting amount of options to be found in the `rsync` manpage.  An important point to remember is what no-slash and trailing slashes on directories mean.

For instance if I want to copy everything within a folder named `/home`, I would use `rsync -av /home/ /path/to/destination` to copy the files within `/home`, to the folder `/path/to/destination`.

On the other hand, if I want to copy the folder `/home` itself to `/path/to/destination`, then I would use `rsync -av /home /path/to/destination`, which will then create the folder `/path/to/destination/home`.

Here are a few of the most important command-line options to remember.

* `-v`: verbose, will tell you what file its on, how many left to check, etc.
* `-a`: archive, will set most of the preferable options, this is shorthand for -rlptgoD. If in doubt, you most always want the -a option.
  * `-r`: recursive
  * `-l`: copy symlinks as files, not the file to which they point.  Why do you want this? It prevents your rsync job from looping back to a directory above itself, causing an infinite loop, in the case there are a few unruly symlinks in your filesystem tree.
  * `-p`: preserve permissions
  * `-t`: preserve modification timestamps
  * `-g`: preserve group ownership. rsync is smart enough to change the group ID by name, rather than numerical group ID on the destination.
  * `-o`: preserve ownership. Same behavior applies to user ID by name, rather than numerical ID.
  * `-D`: preserve special files. Such as device files, named pipes, etc.

Some other useful switches

* `--progress`: gives you per-file data transfer rates, and spinning progress bars if you\'re into that sort of thing.
* `--stats`: gives you a nice summary rate, and speed-up rate at the end of the job.

## Set up a rsync daemon

If you are cloning a database or migrating a complex ERP system, the fastest way to do that is export a root share for your destination host, by defining it in rsyncd.conf. I typically set up a temporary read-only rsync daemon on the production host.  The reasoning for that is the the client end of rsync will consume more memory as it indexes the list of files to transfer.  Therefore rsync should have less impact by running the job from a pre-production host, while the daemon runs on your production server.

Anyway, the configuration files for the temporary rsync daemon on your production system should look something like this.

{% highlight bash %}
    #/etc/rsyncd.conf
    syslog facility = local3
    read only = yes list = yes
    auth users = root
    secrets file = /etc/rsyncd.secrets
    hosts allow = 1.2.3.4/32
    uid = 0
    gid = 0

    [root]
    comment = /
    path = /
{% endhighlight %}

Set a password for the daemon in /etc/rsyncd.secrets, and set the owner/group of that file to root:root with 600 permissions.  The password file format looks like this.

{% highlight bash %}
    #/etc/rsyncd.secrets
    root:SuperSecretPasswordFTW
{% endhighlight %}

Open TCP port 873 for the pre-production host, an appropriate iptables rule would look something like this.

{% highlight bash %}
    iptables -I INPUT -s preprod-server -m tcp -p tcp --dport 873 --syn -j ACCEPT
{% endhighlight %}

Finally run the following command to start up your temporary rsync daemon on the production host.

{% highlight bash %}    
    rsync --daemon
{% endhighlight %}

Now you should have all the components necessary to copy files over the network using the rsync protocol, an rsync client on the pre-production server, and an rsync server on the production server.  Your command should look something like this.

{% highlight bash %}
    rsync -av root@prod-server::root/home/ /home
{% endhighlight %}

Yet another way of saying the same thing, but also copying any special permissions on the top-level directory of home to the /home directory on the pre-prod server.

{% highlight bash %}
    rsync -av root@prod-server::root/home /
{% endhighlight %}

The rsync url components are user (root), followed by an @, hostname (prod-server), followed by a separator (::), followed by the share name (root), followed by the source directory with, or without an optional trailing-slash.

Once the rsync job has finished on the pre-production server, you can run the following to kill off the daemon, and remove your temporary iptables rule.

{% highlight bash %}
    killall -9 rsync
    iptables -D INPUT -s preprod-server -m tcp -p tcp --dport 873 --syn -j ACCEPT
{% endhighlight %}

## Advanced rsync usage

### block-level replication

In theory you would want to use the following option to update block-level changes on large files.

* `--inplace`: rsync writes updated data directly to a file, instead of making a copy and moving it into place.
  * I strongly advise you to RTFM on this one, it comes with several warnings.  But from what I understand, this option was added to rsync for cloning large files such as databases.  Both server\'s rsync programs need to support the in-place option for this to work, otherwise it will give you an error.

I usually kick off a large rysnc job on a hot database, and run one or even a few catch-up jobs on a cold database before the final switch in any large-scale data migration.  What that means is you can run rsync on a running database, but you will not get a consistent Oracle DBF file on your pre-production host during a hot run.  It doesn\'t matter if you use the `--inplace` option, or not, on the first hot run.  Either way, you probably will not get a consistent copy of the database.

However, on any subsequent cold run (when the database is shut down) `--inplace` can significantly reduce the amount of data written on the pre-production host.  Because this particular option has the ability to do block-level updates to large files, when used on your final rsync catch-up job.  I have witnessed rsync updating half a Terabyte worth of dbf files in close to half an hour, during a final cold run on a quiet weekend.  I have also seen it take up to four hours to do just as much after close of business on a Friday night.  Your mileage may vary from my results, probably the most significant variable in rsync performance is the amount of usage between the initial hot run and final cold run.

### Extending rsync by automation with Bash

Lets say you have a long paired list of source and destination directories.  For example, I might want to archive `/home/u/user1` on the prod server to `/home/user1` on the pre-prod server.  The rsync rules of the trailing slashes on directories do come into play here.

So after playing around with some perl/sed, and awk to generate a list of source and destinations from a password file on the production server.  I might have a file rsync-home.list containing syntactically correct paired rsync sources and destinations, which would look something like this.

{% highlight bash %}
    root@prod-server::root/home/a/alice /home
    root@prod-server::root/home/b/bob /home
    root@prod-server::root/u01/app/oracle /u01/app
    root@prod-server::root/home/c/charlie /home
    root@prod-server::root/home/d/dave /home
    root@prod-server::root/home/e/eve /home
    root@prod-server::root/home/m/mallory /home
    root@prod-server::root/home/t/trent /home
    root@prod-server::root/home/w/walter /home
    root@prod-server::root/usr/local/vendor /usr/local
{% endhighlight %}

Unfortunately there is no switch for rsync to read such a list, and operate on it directly.  However, it is fairly easy to script with a bash one-liner, like so.

{% highlight bash %}    
    RSYNC_PASSWORD="SuperSecretPasswordFTW"; cat rsync-home.list | while read LINE; do rsync -av --progress $LINE; done
{% endhighlight %}

The `RSYNC_PASSWORD` is an environment variable you can set to avoid having to type in the actual password for the rsync daemon, just be aware it will show up in your .bash_history file, and output from `ps -ef`.  So it is advisable to not use the same password as your root account for the purposes of rsync.

### Hunting down symlinks in the archive tree

Remember when I said you usually don\'t want to copy what a symlink points at, rather copy the pointer (symlink file) itself.  If you want a quick and easy way to make a list of symlinked directories within the `/u01` tree on your production server, you can pair `find`, and `awk` commands to generate a list like this. The `-type l` option tells `find` to look for symlinks, the `-xtype d` option tells find to look for symlinks pointing at directories.  The `$NF` variable in `awk` returns the last field from the long-list `ls -l` output.

{% highlight bash %}
    find /u01 -type l -xtype d -exec ls -l {} ; | awk '{ print $NF }'
{% endhighlight %}

Which returns a handy little list of items remaining to be copied over to our pre-prod server by rsync.  The couple of returned entries without a leading-slash can be thrown out, those are links within the `/u01` tree and would therefore be copied as a result of running an rsync job on `/u01`.

{% highlight bash %}
    /u04/app/oracle/RMAN
    /u03/app/oracle/arclogs
    /u05/app/oracle/exports
    /u02/app/oracle/oradata/fooprod
    /u06/app/oracle/oradata/foopreprd
    client
    /u08
    linux
{% endhighlight %}
