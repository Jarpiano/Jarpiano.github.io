---
title: "Sneaky Write-Up"
date: 2025-07-06T21:47:27-05:00
draft: false
author: Jarpiano
tags:
    - WriteUp
image: /images/sneaky-banner.png
description:
toc:
center: true
---
# Sneaky (Medium) Box
Sneaky is a retired box focusing on IPv6 connections and buffer overflow vulnerabilities. 
## Initial Foothold
Enumerating through the box, we see there is an HTTP server available. Let's fuzz this and see if we can find anything. After fuzzing, we see that there is a /dev page that leads to a login screen. We can try out some payloads
		  
When entering a `'` in the Username section, we get an `Internal Server Error`. This usually occurs with SQL databases. Putting in a classic `' OR 1=1; -- -` payload let's us access the logged on page. On that page we find a RSA key we can use for an SSH login. You can gain a foothold on the database via a SQL injection available on their login page. 

On using NMAP, we don't find any open SSH ports `sudo nmap <ip> -sS -Pn -n --disable-arp.` Let's try it with the ` sU` UDP flag and put a limit on the ports, 
```
sudo nmap <ip> -sU -Pn -n --disable-arp -F
```
We get two open ports! `dhcpc` and `snmp` are available.

Based on the challenge description, we should enumerate SNMP. `snmpwalk` is a great option, but `Metasploit` includes a detailed and clean `snmp_enum` module in `msfconsole`. Let's use that.

I didn't really get anything from this except for a few ports and IPs that I couldn't really access. Upon looking at some forums, it seems that IPv6 may provide us with some clues. I found a good IPv6 SNMP address scanner called `Enyx` available on GitHub. Using that, we're able to find an IPv6 address. Using the `deadbeef`IPv6 address, the `nmap` scan brings us an open 22 SSH port. We can connect to that as normal on SSH: 

```
ssh <found-user>@<ipv6> -i <rsa-key>  -o PubkeyAcceptedAlgorithms=+ssh-rsa
```

`RSA` is usually disabled so here's a quick liner.
  
We've got access on the system! We can find our user flag in the ~ or home directory of our t* user.For enumeration through the machine, we can use a binary such as LinEnums or LinPeas. Both will achieve the same result. On inspection of the output, we'll see that there is a unknown binary `/usr/local/bin/chal` with SUID and SGID properties. Looking at our description, this must be our buffer overflow vulnerability!

## Privilege Escalation
  
There are several ways to approach this. I used `ghidra` to decompile and see what's happening underneath. On inspection of the main method, we find that there is a potential buffer overflow with a char array of size 362 that uses the vulnerable `strcpy()` method. When we input into the method, we do see that we get a segmentation fault on inputs greater than 362. When we go into `gdb`, we also find that we're overwriting the EIP.
  
We can construct a payload that executes with root privileges using this vulnerability. Grabbing a shell payload from shell storm, we're able to gain privileges through the following: First we run a python script in the input that can be executed, this will look like:

```
$(python  c "print(<data>)")
```
	  
In this Python script, we can include our payload. First let's put in our payload, keeping in mind the size, in bytes, of our web shell.Then we can create our NOP slide that is 362, the size of our buffer, minus the size of our payload.Then we need to find our address to overwrite the EIP with to execute our code.
		  
To do this, let's place arbitrary hex values in the EIP at the moment. Executing our NOP slide + payload + EIP override in `gdb`, we see that we get a segfault and our hex values are overwriting our EIP. Let's set up a breakpoint at main using `b main` and rerun our Python code. To continue on, we can do `disas main` to find the next place we should breakpoint which is the address after the `strcopy()` method. Another way to find this isto step by instruction using `ni` to see the instructions that occur before our segfault. We then set up a breakpoint using `b *<address>.` The next part here is crucial. We need to find an address in memory that **DOES** execute. We can't just use any address in memory as that address could bereserved for stack variables and other overhead. I didn't find a lot of notes on this, so just pick an address in the NOP sled for now.

Once you do that you now have all 3 components. Your final python code should be:

```
$(python -c "print(b'\x90'*<size> + b'<PAYLOAD>' + b'<ADDRESS>)
```

You can execute this outside of GDB as well by simply doing: `/usr/local/bin/chal $(PYTHON CODE)`. Once you execute, you should get a root shell and you can find the flag in the /root directory. If you keep getting a segfault, go back into GDB and pick another address in the NOP sled. Do this again and again til it works basically.

**Fin.**