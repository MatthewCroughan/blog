+++
title = "Lustrating My Frustrating Arch Install (De-Antergosifying)"
date = 2020-11-11T23:25:09Z
cover = "./defenestratergos.webp"
tags = ["linux", "arch", ""]
keywords = ["linux", "nixos", "arch"]
description = ""
author = "Matthew Croughan"
authorTwitter = "matthewcroughan"
showFullContent = false
+++

> lustrate  
/ˈlʌstreɪt/  
*verb*  
  purify by expiatory sacrifice, ceremonial washing, or some other ritual action.
  *"a soul lustrated in the baptismal waters"*

# The Past (Thu 21 May 23:26:00 GMT 2020)

2 years ago, in 2018, I began using Linux for real. My distribution of choice
was [Antergos](https://en.wikipedia.org/wiki/Antergos), an 'easy button' Arch
installer that was really popular at the time. The first package I added
post-install apparently was `synergy`, a package to share my mouse/keyboard with
my still-existing Windows 10 machine:


```
matthew@thinkpad ~ $ head -n 3 /var/log/pacman.log
[2018-05-21 23:26] [PACMAN] Running 'pacman -S synergy'
[2018-05-21 23:26] [ALPM] transaction started
[2018-05-21 23:26] [ALPM] installed synergy (1.8.8-3)
```

Little did I know that by choosing to use Antergos, I had already **infected my
future.**

## The Now (Thu 12 Nov 05:38:30 GMT 2020):

It's 5AM. I have work tomorrow. What better way to spend my time than debugging
this crap? I love it. 

{{< code language="PACMAN" title="Full Output" id="1" expand="Show" collapse="Hide" isCollapsed="true" >}}
matthew@thinkpad ~ $ sudo pacman -Syu
:: Synchronising package databases...
 core is up to date
 extra is up to date
 community is up to date
 multilib is up to date
:: Starting full system upgrade...
:: Replace pygobject-devel with extra/python-gobject? [Y/n] y
:: Replace user-manager with extra/plasma-desktop? [Y/n] y
:: Replace xorg-luit with extra/luit? [Y/n] y
resolving dependencies...
looking for conflicting packages...
error: failed to prepare transaction (could not satisfy dependencies)
:: removing xorg-luit breaks dependency 'xorg-luit' required by antergos-common-meta
:: removing user-manager breaks dependency 'user-manager' required by antergos-kde-meta
{{< /code >}}

`:: removing xorg-luit breaks dependency 'xorg-luit' required by antergos-common-meta`

`:: removing user-manager breaks dependency 'user-manager' required by antergos-kde-meta`

*Agony.*

[When Antergos discontinued in May
2019](https://itsfoss.com/antergos-linux-discontinued/), I removed their mirrors
from `/etc/pacman.conf`, which should have been enough. Everything had been fine
until now.

#### The problem?

1. The now dead and unmaintained
[meta-package](https://wiki.archlinux.org/index.php/Meta_package_and_package_group)
is causing [dependency hell](https://en.wikipedia.org/wiki/Dependency_hell) since
I did not remove it when Antergos died.

2. My decision to use Antergos has screwed me 2 years later.

3. I should have moved over to NixOS a long time ago.

#### The solution?

1. Simply remove the meta packages via `pacman -Rdd` 

```
matthew@thinkpad ~ $ sudo pacman -Rdd antergos-kde-meta antergos-common-meta

Package (2)           Old Version  Net Change

antergos-common-meta  1.5-1          0.00 MiB
antergos-kde-meta     1.1-1          0.00 MiB

Total Removed Size:  0.01 MiB

:: Do you want to remove these packages? [Y/n] 
```

#### The result?

🕒 1 wasted hour. 

I did not know what meta-packages were, or the consequences of removing them.

I can now update my Arch system. Hopefully there is no more Antergos cruft still
lingering in my system.

Everything will be fine going forward, as long as:

* Power isn't lost during upgrade
* I do not `^C` the upgrade at an inopportune time
* Creation of the initramfs succeeds ([it's weird
  sometimes...](https://bugs.archlinux.org/task/54918)) 
* I do not run out of disk space

If any item from this list isn't met, the system can fail to boot. Since updates
are not atomic in any way. The only way to recover from such a situation is
`arch-chroot` on an external recovery boot image such as the Arch install media
itself.

This has happened to me too many times on this install, I count a total of 5
times. Though, if anything, this is a testament to how recoverable bad
situations can be on Linux if you have the
[know-how](https://docstore.mik.ua/orelly/unix3/upt/ch14_03.htm)

Stuff like this really makes me want to move to NixOS, where these issues don't
exist, or at least simple rollback from failure states is easy and built in.
