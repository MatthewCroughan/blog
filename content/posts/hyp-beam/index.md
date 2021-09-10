+++
title = "Hypercore Beam - Break Through NAT With This One Weird Trick"
date = "2021-08-24T01:51:58+01:00"
author = "Matthew Croughan"
authorTwitter = "matthewcroughan" #do not include @
cover = "hyperzfs-clickbait.png"
tags = ["linux", "nixos", "zfs","hypercore"]
keywords = ["linux", "nixos", "zfs","hypercore"]
description = ""
showFullContent = false
+++

`hyp beam` can be thought of as a drop-in replacement for `nc`, for those
situations where you just can't be bothered to set up a VPN or manually
establish a path between two machines via traditional methods like port
forwarding. It provides the user with a method of breaking through NAT and
transferring data via the Hypercore protocol between two machines, peer to peer,
anywhere in the world. Peer discovery happens via DHT/mDNS using a concept
called [Hyperswarm](https://hypercore-protocol.org/guides/modules/hyperswarm/),
not too dissimilar from what Bittorrent clients have always done; although its
usage of mDNS is quite unique and cool.

Hypercore Beams are end to end encrypted. There's no overhead that I can
measure, and it can be used directly in place of netcat. IP Address, network
conditions, network port, all cease to matter since with `hyp` since we only
care about piping data between processes. A shared secret you pass to the
command which can be as simple as a set of words is used as an identifier, in
place of an IP address. So what are some of the things I use it for?

# Sending ZFS datasets around the internet

![image](/hyperzfs.png)

Usually, `zfs send` requires LAN connectivity, or a VPN, which is a pain to
think about. Why think about where your machines are? With `hyp beam` I can just
push and pull ZFS datasets through an encrypted tunnel without caring where I am
in the world.

```
## Machine 1
zfs send eggshells/data | hyp beam "secret words"
## Machine 2
hyp beam "secret words" | zfs recv zpool/data
```

Machine 1 pipes `zfs send` into `hyp beam` which chooses a secret. This secret
can be any string in `hyp beam "<secret>"`. This secret is then used as an
identifier, it is all you need to establish communication between the two
machines. Machine 2 then pipes the output of `hyp beam "<secret>"` into `zfs
recv`. How simple is that?

Sidenote: I call one of my pools `eggshells` because often whenever I interact
with it, it's like [walking on
eggshells.](https://dictionary.cambridge.org/dictionary/english/walk-be-on-eggshells)

# Video streams over the WAN

![image](/hypermpv.png)

Sending Video over `hyp beam` is surprisingly practical. It's a good
demonstrator of how `hyp beam` can be used in place of `nc`. Once you realize
that this is just a peer to peer link to send data through with no overhead, it
ceases to be impressive. The ease of use is what is amazing.

The following will use fancy bash wankery to create two FIFO pipes with which to
transmit video data from `ffmpeg` through. Such that two clients can watch the
video via Netcat, or anywhere in the world via Hypercore.

## With Netcat

This requires you to know the IP address of the machine, and have a direct
connection via LAN/VPN.
```
## Sending
ffmpeg -vcodec mjpeg -i /dev/video0 -f avi - | tee -- >(nc -l 1234) >(nc -l 5678) > /dev/null; wait

## Recieving
# Client 1
nc 127.0.0.1 1234 | mpv -
# Client 2
nc 127.0.0.1 5678 | mpv -
```
## With Hypercore Beam

This does not require knowing the IP address of a machine, or having a direct
connection via LAN/VPN. It will work between two machines anywhere in the world
and is likely to succeed even when it encounters firewalls. The port also does
not matter, because we only care about the data being piped into a process.

```
## Sending
ffmpeg -vcodec mjpeg -i /dev/video0 -f avi - | tee -- >(hyp beam "1234") >(hyp beam "5678") > /dev/null; wait

## Recieving
# Client 1
hyp beam "1234" | mpv -
# Client 2
hyp beam "5678" | mpv -
```

# Sharing files with UNIX friends

Sometimes I want to share folders or files with friends that are on macOS/Linux.
Usually I would use `magic-wormhole` for this, but one of the annoying things
about that program is that if you want to send a folder it first has to be
zipped. That `zip` process can take a very long time and requires you to have as
much RAM as the filesize you are trying to send. It also doesn't even give you
as good of a compression ratio as gzip. Since `hyp beam` only cares about piping
data, we can make up our own method of sending the folder, by using `tar`.

```
# Sender (Sending the Folder)
tar cf - <folder-name> | hyp beam "secret"

# Receiver (Receiving the Folder)
hyp beam "secret" | tar xvf -
```

You could easily invent shell aliases such as `send-to-matt` or `receive-matt`
ready to pack/unpack files, hiding the `tar` command from sight such that this
feels like a proper application for sending and recieving files rather than
having to remember the `tar` syntax each time.

# What will you use it for?

https://hypercore-protocol.org/protocol/
