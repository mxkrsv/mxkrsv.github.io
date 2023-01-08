---
title: "postmarketOS on Orange Pi 3"
date: 2021-09-10T16:30:33+03:00
tags: ["linux", "alpine", "postmarketos"]
---

What can I say after debugging the kernel and the bootloader for several days?
Nothing other than Allwinner CPUs are shit with shitty mainline support.

But yet, it kinda works.

<!--more-->

Kinda?

Yeah, excluding shitty onboard PCIe.

However, with that exact shitty board for PHY to work a specific power-on
sequence is needed, which can be performed correctly only by the kernel.

So, in addidion to patches from [Megi's linux tree](
https://github.com/megous/linux) and from [pmaports](
https://gitlab.com/postmarketOS/pmaports/-/tree/master/device/main/linux-postmarketos-allwinner
), to have Ethernet working, you have to disable regulator setup in
arm-trusted-firmware, which requires building it separately specifically for
that board.

Not that bad giving that a short time ago you would had to patch
arm-trusted-firmware to make it not touch any regulators, but still sounds
unfortunate, huh? That's it, so next time don't try to save money and buy a
decent non-Allwinner SBC.

So it goes.
