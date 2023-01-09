---
title: "Ambassador (medium)"
date: 2023-01-08T06:29:46+03:00
tags: ["hackthebox"]
---

<!--more-->

Run nmap, find Grafana on port 3000.

Find out that it's vulnerable to unauthorized arbitrary file read attack
([CVE-2021-43798](https://nvd.nist.gov/vuln/detail/CVE-2021-43798)).

Exploit that CVE (see [this](https://github.com/jas502n/Grafana-CVE-2021-43798)
or [this](https://github.com/pedrohavay/exploit-grafana-CVE-2021-43798) as
references).

... So, first time I passed that machine incorrectly cause someone created
another (easier) hole, so I'll rewrite that when I pass it again.
