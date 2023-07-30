---
title: "postmarketOS on Orange Pi 3"
date: 2021-09-10T16:30:33+03:00
tags: ["linux", "alpine", "postmarketos"]
---

Somehow I found myself highly dissatisfied with Armbian,
and so I came up with an idea to port postmarketOS to my SBC.

Soon I realized that cheaping out on an SBC was indeed a bad idea.

<!--more-->

Problems begin in the very beginning:
despite kernel booting up just fine,
I needed to patch it just to get the SD/MMC to work.
Realization of that issue alone already took one evening.

Well, OK, we got the system up and running. What else?

Suddenly, Ethernet doesn't work.
Why?
It took another two or three evenings to realize that the problem is actually
in the arm-trusted-firmware[^1], and also to come up with a working patch.
Armbian works, of course, but their sources and patches tree is surely something,
so it was easier to look at LibreELEC and do as they do.

Oh, and I lied in the very beginning,
the kernel that "boots but without storage support" is also not mainline,
but is a [Megi's linux tree](https://github.com/megous/linux).

So, to sum up, to make everything *that can work* to work, you have to:
- Use the Megous tree.
- Apply additional patches from [pmaports](
https://gitlab.com/postmarketOS/pmaports/-/tree/master/device/main/linux-postmarketos-allwinner
).
- Build a separate incompatible ATF just for that board.

On, is there something that *cannot work*, you will ask?
Sure!
- The onboard PCIe won't work by design
because of the hacky controller that requires address wrapping.
- The audio chip doesn't have a driver.
- There's no support in U-Boot for showing boot logs on HDMI.
- Maybe something I didn't care about.

A conclusion is already in the preamble: just go buy an SBC with a Rockchip CPU.

[^1]: That exact board's PHY requires a specific regulator power-on sequence
that can only be handled by the kernel, therefore ATF powering it on breaks everything.
Until the recent, making ATF leave regulators alone required patching it.
