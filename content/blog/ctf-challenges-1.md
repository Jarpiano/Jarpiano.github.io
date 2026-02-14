---
title: "CTF Challenges 1st Edition"
date: 2026-02-13T21:36:50-06:00
draft: false
author: Jared Baron Panares
tags:
image:
description:
toc:
center: true
---
# CTF Challenge Samples - Edition 1
These cover somve of the CTF challenges I developed during my time operating
under the ISSS organization at UT Austin. The OSINT challenges in particular,
one of my main departments, will be mostly skipped for TOS concerns.

## Open-Source Intelligence (OSINT)
Though I can't go into too much detail, I'll go through the setup
methodology as much as I can. Typically, this followed an scavenger workflow.
The bulk of finding the flag would be on social media or forum sites with
a more lenient email policy. Connecting multiple accounts across these sites
helped create a more realistic web of connections for students, helping them
develop skills in Recon. I recommend the usage of SimpleLogin for these
purposes.

## Reverse Engineering
### Hidden Contents
This covered basic Javascript deobfuscation. When run, the student would be
given a prompt. This was a red herring. The student needed to run the 
Javascript through deobfuscator.

[Hidden-Contents tar.gz File](/challenges/hidden-contents.tar.gz)


## Networking
### Within Plainsight
Within Plainsight has you look through an Apache log file and determine the
name of an .aspx webshell file that was uploaded. The Apache log file was
sourced from a publicly available Github providing log file samples, and
slightly edited to contain .aspx URL artifacts for the purposes of the CTF>

[Within-Plainsight tar.gz File](/challenges/within-plainsight.tar.gz)

## Cryptography
### Dracula's Message
Dracula's Message is a Halloween-themed challenge covering hex encoding
and hashes. The student decodes the hex message and is able to obtain a hash.
The hash can be broken through hashcat, using the rockyou.txt file, or online
hash decryptors like crackstation. The cracked password can be used for the
encrypted file provided.

[Draculas-Message tar.gz File](/challenges/draculas-message.tar.gz)

## Pwn (Binary Exploitation)
### Frankenstein's Key
Frankenstein's Key takes two routes on Binary Exploitation. A student could
either use a dynamic approach by manipulating the if statement's values. This
could be done through gdb or a more specialized program like pwndbg. Another
option would be to use Ghidra to obtain the flag. The flag is encoded in base64
and is split across 20+ lines.

[Frankensteins-Journal tar.gz File](/challenges/frankensteins-journal.tar.gz)
