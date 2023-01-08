---
title: "Ambassador"
date: 2023-01-08T06:29:46+03:00
tags: ["hackthebox"]
---

There is another (maybe more fair) way via Grafana's db and MySQL, but here is
the simpler one that brings us straight to root leaving the user and privesc
behind:

<!--more-->

Run nmap, find Grafana on port 3000.

Find out that it's vulnerable to unauthorized arbitrary file read attack
([CVE-2021-43798](https://nvd.nist.gov/vuln/detail/CVE-2021-43798)).

Exploit that CVE (see [this](https://github.com/jas502n/Grafana-CVE-2021-43798)
or [this](https://github.com/pedrohavay/exploit-grafana-CVE-2021-43798) as
references).

Read the `/etc/passwd`:
```
root:x:0:0:root:/root:/bin/bash
root2:WVLY0mgH0RtUI:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
...
```

Notice that the `root2` user has a plain
[`crypt(3)`](https://man7.org/linux/man-pages/man3/crypt.3.html) hash (aka just
DES which is easy to bruteforce).

Brute it with `john` and find the password (it's not even in [`rockyou.txt`](
https://github.com/danielmiessler/SecLists/blob/master/Passwords/Leaked-Databases/rockyou.txt.tar.gz
), but `john`'s incremental ascii is fast enough for DES).

Use it to log into the machine via ssh and get flags for both root and user.

Done!
