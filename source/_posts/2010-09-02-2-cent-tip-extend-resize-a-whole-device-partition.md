---
layout: page
title: 2 Cent Tip - Extend (resize) a whole device partition. 
date: 2010-09-02
comments: true
categories:
- storage
- Enterprise Linux
---

Occasionally I have to resize partitions on iSCSI or Fiber-Channel attached SAN storage.  Both technologies allow you to easily extend the available storage for a host by extending LUNs, or volumes.  A common problem after extending the size of the LUN, or volume, is resizing partitions to fill out the new size. 

For the most part, I usually fire up [PartedMagic](http://partedmagic.com) and its a snap, even with Fiber-Channel attached enterprise storage.  Once the HBAs (Host Bus Adapters) have been zoned to Fiber-Channel switches, then the HBAs do all the heavy lifting.  In other words on Fiber-Channel, it doesn't matter if you're using PartedMagic, or [Knoppix](http://www.knopper.net), the server just knows where the storage is and whether it is in an attached state.  The only dependency for this working on a Live boot disk are drivers for the HBA cards.

iSCSI is a bit different.  Because, iSCSI relies on commodity Network Interface Cards, this technology is largely implemented in software.  One perceived advantage is iSCSI may seem less complicated to use than Fiber-Channel storage.  Unfortunately, in this case, PartedMagic did not have open-iscsi software, and I could install open-iscsi in the Knoppix Live ramdisk. However, because Knoppix came with an outdated iSCSI kernel module, it was not new enough to inter-operate with the open-iscsi software.

Furthermore, the version of [parted](http://www.gnu.org/software/parted/index.shtml) that shipped with RHEL 5 threw an incompatible filesystem error, refusing to modify the filesystem. So, in the end, I twiddled some bits on the partition table with fdisk, and used resize2fs to extend the partition.

Assuming you have a backup of the filesystem you are working on, you can proceed with the following steps to extend a single partition to the end of the extended volume.  If you have multiple partitions on a volume, you may want to stick to more reliable methods of resizing and extending.  If you screw up the cylinder boundaries on a device with multiple partitions, you'll definitely lose data.  A single partition, in this example, is a much simpler scenario.

The device name in this example is `/dev/mapper/u02`, the first partition is `/dev/mapper/u02p1`:

Run `fdisk -l /dev/mapper/u02` to get the starting cylinder.

{% highlight bash %}
    root@localhost:~# fdisk -l /dev/mapper/u02
    Disk /dev/mapper/u02: 100.9 GB, 108340550042 bytes

    255 heads, 63 sectors/track, 13171 cylinders
    Units = cylinders of 16065 * 512 = 8225280 bytes

              Device Boot      Start         End      Blocks   Id  System
    /dev/mapper/u02p1               1       13171   26450329   83  Linux
{% endhighlight %}

Reboot the server after extending the volume or LUN on your SAN.  You want to do this before extending the partition in your Operating System.  The Operating System needs to re-read sector 0 on the extended SAN volume, before continuing. Note, that fdisk will report **26109** cylinders instead of **13171**, after rebooting the server.

Next, we will run: `fdisk /dev/mapper/u02`, and then hit the keys: `d, n, p, 1, [enter], [enter], w`

{% highlight bash %}
    root@localhost:~# fdisk /dev/mapper/u02
    WARNING: DOS-compatible mode is deprecated. It's strongly recommended to
             switch off the mode (command 'c') and change display units to
             sectors (command 'u').

    Command (m for help): d
    Selected partition 1

    Command (m for help): n
    Command action
       e   extended
       p   primary partition (1-4)
    p
    Partition number (1-4): 1
    First cylinder (1-26109, default 1):
    Using default value 1
    Last cylinder, +cylinders or +size{K,M,G} (1-26109, default 26109):
    Using default value 26109

    Command (m for help): w
    The partition table has been altered!

    Calling ioctl() to re-read partition table.
    Syncing disks.
{% endhighlight %}

Finally, run `resize2fs` on `/dev/mapper/u02p1`. If you are using ext3, you can do an on-line resize while the volume is mounted.  It is probably safest to `umount` the partition to be re-sized, however.

{% highlight bash %}
    root@localhost:~# resize2fs /dev/mapper/u02p1
    resize2fs 1.39 (29-May-2006)
    Filesystem at /dev/mapper/u02p1 is mounted on /u02; on-line resizing required
    Performing an on-line resize of /dev/mapper/u02p1 to 52430127 (4k) blocks.
    The filesystem on /dev/mapper/u02p1 is now 52430127 blocks long.
{% endhighlight %}

Refer to the `resize2fs` for more information on the command, and its proper usage.
