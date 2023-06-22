---
title: "How I accidentally migrated my desktop to NixOS"
date: 2023-06-22T01:31:35+03:00
tags: ["linux", "nixos", "alpine"]
draft: true
---

I have been using Alpine Linux for about a year or more.
<!--more-->
As stated [on its site](https://www.alpinelinux.org/about/),
it gives you "crystal-clear Linux environment",
and that pretty much describes the experience.

It was mostly enjoyable, but I faced small issues here and there,
like outdated documentation or lack of one at all,
unfair distribution of scarce manpower
(packages are fine, but most other projects are stalled),
etc.

By the way, there is [my NixOS configuration](https://github.com/mxkrsv/nixos-config),
as well as [old generic dotfiles](https://github.com/mxkrsv/dotfiles).

## Why?

Adding a new level of abstraction is a natural idea if something isn't as straightforward
as one would want it to be.
Every reason here is basically about noticing a gap for another level of abstraction that
NixOS successfully fills.

### 1. Dotfiles

I felt that my Alpine setup was kinda hacky.

I used [Drew's way](https://drewdevault.com/2019/12/30/dotfiles.html) for managing dotfiles.
I believe it to be the best way out there that doesn't require
relying on specific dotfile managing programs, none of which I liked.

Doing that already felt pretty wrong: the entire `$HOME` has to be a git repo!

More than that, I also wanted to manage my secrets like regular configs,
especially given that secrets are actually configs.

I came to creating a second repo, `private_dotfiles`,
placing it to `$HOME` along with regular dotfiles,
and tracking secrets in that repo.
I don't think I have to tell how dirty that method is, however,
it's the best one I could think of.

### 2. System deployment

The more I needed from the system,
the more manual actions were required to achieve that.
The larger the effort put into configuring a system is,
the simpler it is to overlook something important.

E.g. just to set up [Sway](https://swaywm.org/) I would need to do something like that[^2]:
```
# setup-devd udev
# adduser mxkrsv input
# adduser mxkrsv video
# apk add seatd
# rc-update add seatd
# rc-service seatd start
# adduser $USER seat
# apk add sway foot
$ cat >~/.profile <<EOF
if test -z "${XDG_RUNTIME_DIR}"; then
  export XDG_RUNTIME_DIR=/tmp/$(id -u)-runtime-dir
  if ! test -d "${XDG_RUNTIME_DIR}"; then
    mkdir "${XDG_RUNTIME_DIR}"
    chmod 0700 "${XDG_RUNTIME_DIR}"
  fi
fi
EOF
```

Needless to say the amount of manual intervention required is significant,
and that such intervention is required to get working every part of the system you want
to get working[^3].

The resulting system is crystal-clear, because everything that present is present
just because you did it that way, but everything comes at a cost.

### 3. Moving parts

Continuing what was put up in System deployment section:
the system is mutable and there's just too many moving parts,
though somewhat less than in a typical distro.
Something can and will break down eventually one day or another.
There's also no way to tell in which exactly state the system currently is
without checking everything manually.

### 4. Packages

Some things were pretty hard to package for Alpine, like `sqlmap`.
Some packages ended up operating incorrectly at the most inopportune moment.

## What I got

In the ["NixOS in Production" book](https://leanpub.com/nixos-in-production),
the following is said:
*"If you like Alpine Linux then you’ll love NixOS".*
It's damn truth.
I liked how lean Alpine is,
and I love how pure NixOS is.

In some way, it is really similar to Alpine: if something is present,
it's just because you configured your system like so.
The difference here is the *configuration* versus *intervention*.

I noticed that I like to package everything and immobilize as many moving parts
as possible far before I even thought of migration.

### 1. Dotfiles

That problem is solved by [home-manager](https://github.com/nix-community/home-manager)
coupled with [agenix](https://github.com/ryantm/agenix).

I found managing secrets with agenix very simple and convenient.
I also tried [sops-nix](https://github.com/Mic92/sops-nix),
but found it far too complex (particularly relying on highly complicated sops).

### 2. System deployment

My system and my configurations are fully contained in a single git repository.
That is something I could only dream of,
but with [Nix flakes](https://nixos.wiki/wiki/Flakes) it's real.

Both deploying my system configuration and my user configuration are just
one single command each:
```shell-session
$ home-manager --flake . switch
# nixos-rebuild switch --flake .
```

### 3. Reproducibility and consistency

After deploying, you can be sure about your system's state.
Consistency is also about guaranteed:
most things in the system are read-only and are just links to derivations.

All configuration build inputs are pinned to commit hashes via flake lock;
It's almost guaranteed that the fresh system built with the same inputs will be exactly the same.

### 4. Packages

NixOS just shines here: it's top 1 for package count and top 1 (of *real* distros)
for up-to-date projects on [Repology](https://repology.org/repositories/statistics/newest).

If you see some minimally viable piece of software,
it's highly likely that it's already in [Nixpkgs](https://github.com/NixOS/nixpkgs).

## Drawbacks

What would I consider the most troublesome about NixOS?

### 1. Hard to learn

Nix was pretty difficult for me to get into,
despite me having some (though minor) functional programming knowledge and
fair desktop Linux experience.

I found [Nix Pills](https://nixos.org/guides/nix-pills/) to be a pretty good intro
to Nix as a language.

It, however, completely lacks overview of Nix3 (still experimental) features like flakes,
even though I find them to be the most interesting about Nix.
[Documentation](https://nixos.org/manual/nix/stable/command-ref/new-cli/nix3-flake.html) is
not half bad, however, docs aren't always the simplest way to get an understanding of a subject.

### 2. Configuration portability

My configuration is no longer portable now.
In a way it has always been, my dotfiles, although I personally used Alpine,
required just a few tweaks to work on any Linux, or even non-Linux *nix.

Now, since I use home-manager[^1], it's no longer possible to use my configuration as-is
without installing Nix and building home-manager derivations.

### 3. Something else

Also, quite frequently pull requests to Nixpkgs are reviewed
for a larger amount of time that one would like them to be.
Not too surprisingly given how many packages and contributors there are, however.
There's also no way to trivially convert it from Nix back to plain config files,
because output configs generated by home-manager are barely human-readable.

[^1]: I could avoid using home-manager, but why yes, if I already use NixOS?
Also, as described above, home-manager was one of initial drivers of interest for me.

[^2]: Well, actually I had `$XDG_RUNTIME_DIR` stuff in my dotfiles,
but deploying dotfiles was far not a single command either.

[^3]: I even thought about using [Ansible](https://github.com/ansible/ansible) for that,
however, that idea was quickly discarded, partially because of how complex Ansible is itself.
