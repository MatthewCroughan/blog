+++
title = "NixOS + the Worst UEFI ever (S5520UR)"
date = 2021-08-15T03:51:17+01:00
cover = "./uefi-consortium.webp"
tags = ["linux", "nixos", "uefi"]
keywords = ["linux", "nixos", "uefi"]
description = ""
author = "Matthew Croughan"
authorTwitter = "matthewcroughan"
showFullContent = false
+++

If you read [the Wikipedia Page on
UEFI](https://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface) you
will notice there are a few things missing.

The first thing that is missing is names. The second is the word 'engineer'. Did
humans make this? Is any one responsible? Who is to blame? UEFI was designed by
a comittee, so no one can be held accountable.

More digging tells me that UEFI was [developed by over 140 technology companies
as part of a UEFI consortium](https://uefi.org/members). Is there any wonder
nobody can decide how to implement it? Especially when Facebook (!?!?) have a
seat at the table. Some implementations hardcode paths to
`/EFI/Microsoft/Boot/bootmgfw.efi`, probably because some of these companies
only care about Windows and screwed it up for the rest of us.

The first machine I ever ran Linux on was an [Intel Server Board
S5520UR](https://ark.intel.com/content/www/us/en/ark/products/36456/intel-server-board-s5520ur.html).
It taught me Linux, I had no experience prior to it, it has taught me a lot of
things, but today it taught me a little bit about UEFI. Intel gets the most
credit for creating UEFI in various articles, yet this board has one of the most
broken UEFI implementations (irony?). For 3 years, I gave up trying to install
anything in EFI mode with this board, thinking it was impossible. But today, I
decided I wasn't content with that, so went on an 8 hour mission, mostly due to
the time it takes to POST!

Some facts:
  - The board takes 2 minutes to initialize and POST, which is much longer than it takes
    to boot Linux.
  - It cost me only £50~ from eBay.
  - It was a cheap way to get 24 threads
  - The processor(s) (Xeon x5675) cost only £10~
  - It probably isn't worth the effort, but whatever.

# The problem

Attempts to install any OS in EFI mode results in the BIOS of the S5520UR
seemingly being unable to see the `.efi` file on the disk, meaning you can't
really boot anything if you try, since it won't show up in the Boot Menu or Boot
Manager of the BIOS.

For some reason, the NixOS Installer Media shows up in the BIOS as `EFI:Corsair
Voyager SliderX2000AHD(Part1,Sig9FB6382F` which is correct. I can't replicate
this behavior via the installation of any OS. Only this installer media seems to
have the correct configuration to be auto-detected by the S5520UR's UEFI
implementation when flashed to a USB flash drive.

Upon a quick Google you can [this post on
Reddit](https://www.reddit.com/r/servers/comments/ar3k6y/trying_to_install_windows_to_an_intel_s5520ur/)
entitled *"Trying to install Windows to an Intel S5520UR board with RAID0 SAS
array. Gets stuck on "Copying files (0%)". Help?"*

> I was able to fix the problem. **Don't try to install Windows in EFI mode only**,
> this causes the freeze. I guess these kind of servers are just too old.

They also gave up. Unacceptable.

So, what can it be? I happen to be trying to install NixOS, which gives me an
advantage in both trying out various bootloader configurations and explaining
what I tried.

# First attempts

First, I try systemd-boot:

```nix
{
  boot.loader.systemd-boot.enable = true;
}
```

[Nix](https://www.dictionary.com/browse/nix). The disk does not appear as a bootable device in the BIOS.

What about systemd-boot, but this time giving it the ability to touch variables in
`/sys/firmware/efi/efivars/`?

```nix
{
  boot.loader = {
    systemd-boot.enable = true;
    canTouchEfiVariables = true;
  };
}
```

This halts the CPU. So what about GRUB?

```nix
{
  boot.loader = {
    efi = {
      canTouchEfiVariables = false;
    };
    grub = {
       efiSupport = true;
       device = "nodev";
    };
  };
}
```

The same result. No `EFI:<disk>` option in the BIOS for the installation. The
CPU still halts if `boot.loader.efi.canTouchEfiVariables = true;`

I saw [this
section](https://nixos.wiki/wiki/Bootloader#Wrangling_recalcitrant_UEFI_implementations)
of the nixos.wiki on "Wrangling recalcitrant UEFI implementations" and decided
to try and copy the `.efi` file for systemd-boot from
`/boot/EFI/systemd/systemd-bootx64.efi` to `/EFI/Microsoft/Boot/bootmgfw.efi`. I
was quite smug and hopeful about this one, but still, it did not work. But it
would have been an awesome hack if it did.

# rEFInd

The NixOS installer comes with a tool called rEFInd when booted in EFI mode. If
I select and use it, it successfully detects my previous attempts, be it GRUB or
systemd-boot, and successfully allows me to boot those bootloaders in EFI mode.
Now it's getting interesting.

# Getting Closer

There is a section in the nixos.wiki that mentions an interesting option that
I've never seen before, called `boot.loader.grub.efiInstallAsRemovable`. I
decide to try it out, but I still get the same result. However, I find [this
pull request](https://github.com/NixOS/nixpkgs/pull/35528) from NixOS
contributor 'samueldr' which reiterates some of my issues with EFI boot. I
believed it was possible that this option might have some impact on it.

I waste no time. I enter the NixOS Matrix Channel and tag @samueldr and describe
the issue exactly. He informs me that that pull request has nothing to do with
my problem, but that my UEFI implementation is buggy if this doesn't work as a
"fallback". It turns out that all UEFI implementations should fallback to a
default EFI program when present; which in this case should be the GRUB I just
installed with `efiInstallAsRemovable = true;`.

# Epiphany

Finally. We discover that there is an option in the BIOS named "Add new boot
option". It turns out that this must be manually invoked and passed an absolute
path to the location of the `.efi` file on disk if you ever hope to boot
anything with EFI.

| Step 1 | Step 2 |
|-------|--------|
|![](/uefi-bios1.webp) | ![](/uefi-bios2.webp) |

So I do it, I label it `maybe-nixos`, I give it a path to the `.efi` file
`\efi\boot\bootx64.efi`. Then, I try to boot it. It boots without the
assistance of rEFInd.

# Takeaway

The S5520UR:
  - Won't use fallback locations
  - Halts the CPU when `/sys/firmware/efi/efivars/` is poked
  - Apparently requires completely manual setup and configuration of the EFI boot via human fingers
  - Is hot trash.

The only documentation I can find for this that could possibly help a normal
user of the motherboard is buried in the [Aptio HPE
documentation](https://techlibrary.hpe.com/docs/iss/proliant_uefi/UEFI_TM_030617/s_adding_boot_option.html)
