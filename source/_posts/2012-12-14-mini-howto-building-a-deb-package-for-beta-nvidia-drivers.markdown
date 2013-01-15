---
layout: post
title: "Mini-HOWTO: Building a deb package for Beta NVidia drivers"
date: 2012-12-14 12:12
comments: true
categories: linux, gaming, steam, nvidia, debian, ubuntu, packaging 
---

Since the first week of this last November, the closed [Linux Steam beta](http://steamcommunity.com/linux)
has been underway.  I think the latest official count of Linux beta
users is somewhere near 80,000 gamers.  That is a pretty impressive
closed beta headcount considering that is more than 1% of the active
5 million users who use Steam daily.  It is really an exciting time
for all the Linux desktop users.  As the introduction of Steam
has encouraged GPU OEMs to improve their Linux video drivers.  Also,
some of the game developers participating in the Linux beta have been
supporting their customers and fixing bugs, even for those not
officially included in the closed beta program.

<!-- more -->

On to the point of this blog post, as I mentioned, the Steam Linux
beta has led to many Linux video driver improvements from major
GPU vendors.  As of this post, there have been several experimental
NVidia driver releases with major bugfixes and performance improvements
for Linux users.  The NVidia experimental driver releases have perhaps
happened so quickly that it isn't feasible to re-package every iteration
for Ubuntu/Debian users.

Read on if you would like to re-package the NVidia experimental releases,
in a `.deb` package for yourself.  The point of using the `.deb` packaging
is to maintain the ability to upgrade to an official Ubuntu/Debian package
using native package management as newer releases become available in
officially supported repositories.

## Experimental NVidia drivers as deb packages

The package maintainer for the nvidia-graphics-drivers has general build guidelines [here](https://github.com/tseliot/nvidia-graphics-drivers/blob/master/debian/nvidia-current.README.Debian).
If you're desperate for the new drivers via Ubuntu packaging, you can rebuild them with the 310.19, or 313.09 packages.
Here is the process I followed to install the latest NVidia Beta drivers from deb packages.  The major version bump
to 313.09 is a slightly different process.  I could not find any automated way to regenerate the names of the packaging templates
to match the version increase.

### Minor version bump 310.14 - 310.19

1. `cd /usr/src` (you'll need src group membership to write to /usr/src)
2. `apt-get source nvidia-experimental-310`
3. `sudo apt-get build-dep nvidia-experimental-310`
4. `sudo apt-get install devscripts`
5. `cd nvidia-current && rm -f *.run`
6. Download both NVIDIA-Linux-x86-310.19.run and NVIDIA-Linux-x86_64-310.19.run.
    - Move those to your build directory, which should be your current working directory.
7. `tar zcvf ../nvidia-graphics-drivers-experimental-310_310.19.orig.tar.gz --exclude debian .`
9. `dch -i`
    - Edit the changelog, making sure to bump the version on the tom line from 310.14 to 310.19. Might get a warning that the source directory changed.
10. `debclean`
11. `debuild`
12. `cd /usr/src` (Your new .deb files should be in /usr/src)
14. `sudo dpkg -i nvidia-experimental-310_310.19-0ubuntu0.1_amd64.deb`

### Major version bump 310.14 -> 313.09

1. `cd /usr/src` (you'll need src group membership to write to /usr/src)
2. `apt-get source nvidia-experimental-310`
3. `sudo apt-get build-dep nvidia-experimental-310`
4. `sudo apt-get install devscripts`
5. `cd nvidia-experimental-310 && rm -f *.run`
6. Download both NVIDIA-Linux-x86-313.09.run and NVIDIA-Linux-x86_64-313.09.run.
    - Move those to your build directory, which should be your current working directory.
7. `tar zcvf ../nvidia-graphics-drivers-experimental-313_313.09.orig.tar.gz --exclude debian .`
8. `dch --package nvidia-graphics-drivers-experimental-313 --newversion 313.09-0ubuntu0.1`
    -  Edit the changelog.  Might get a warning that the source directory changed.
9. `cd debian`
10. `rename 's/(nvidia-experimental)-310(.*\.in)/$1-313$2/' *.in`
    - Rename the `.in` templates for the major version bump.
11. `cd ..`
12. `debclean`
13. `debuild`
14. `cd /usr/src` (Your new .deb files should be in /usr/src)
15. `sudo dpkg -i nvidia-experimental-313_313.09-0ubuntu0.1_amd64.deb`
