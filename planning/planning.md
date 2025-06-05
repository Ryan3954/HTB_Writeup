# 🐼 HTB Machine: Planning
![Screenshot](https://imgur.com/lg7Z2IM.png)
### 🧾 Machine Information
- **IP:** 10.10.11.68  
- **OS:** Linux  
- **Credential:** `admin:0D5oT70Fq13EvB5r`

### 🔍 Information Gathering
- I started with an Nmap scan using the following command: `nmap -A -oN nmap_scan 10.10.11.68`
- This Revealed 2 ports:
![Screenshot](https://imgur.com/M3r2Kp7.png)
- 22 (SSH) and 80 (HTTP)
- Navigating to `http://10.10.11.68` showed a website that didn’t resolve properly at first. I added the hostname to `/etc/hosts`
- Now planning.htb loads in the browser.
![Screenshot](https://imgur.com/f8XMr5c.png)

### 📁 Directory & Subdomain Enumeration
- Initially, browsing the site manually didn’t reveal much, so I used DirBuster to enumerate hidden directories. This yielded several directories and files, but nothing obviously vulnerable.
![Screenshot](https://imgur.com/nI7UjPA.png)
- I then tried Wfuzz to enumerate subdomains using the `services-names` wordlist and discovered one: `grafana.planning.htb`
![Screenshot](https://imgur.com/HcJqYis.png)
- I added it to `/etc/hosts` as well

### 🔐 Accessing Grafana
![Screenshot](https://imgur.com/qq9opF2.png)
- Navigating to the Grafana subdomain showed a login panel. Using the provided credentials: `admin : 0D5oT70Fq13EvB5r`
![Screenshot](https://imgur.com/FFsUAzG.png)
- I successfully logged in. In the settings, it shows that the Grafana version is 11.0.0.
![Screenshot](https://imgur.com/mchYG1R.png)

### 🛠️ Exploiting Grafana (CVE-2024-9264)
- I searched online for any known vulnerabilities for Grafana v11.0.0 and came across: `CVE-2024-9264` a known RCE vulnerability.
- I used the following PoC from GitHub: https://github.com/z3k0sec/CVE-2024-9264-RCE-Exploit
- Running the `PoC` while running `nc` gave me remote shell access.
![Screenshot](https://imgur.com/XLuBP3a.png)
![Screenshot](https://imgur.com/FBAaeQj.pngg)

### 🐳 Detecting Containerization
- While inspecting the environment, I noticed: Presence of the `.dockerenv` file at `/`
![Screenshot](https://imgur.com/8nEeizK.png)
- This suggested I was inside a Docker container.
- I uploaded LinPEAS using wget to enumerate possible escape path

### 🧾 Finding SSH Credentials
- LinPEAS revealed environment variables with credentials: `enzo : RioTecRANDEntANT!`
![Screenshot](https://imgur.com/1W7uvnm.png)
- I used these credentials to try logging in via SSH: `ssh enzo@10.10.11.68`
![Screenshot](https://imgur.com/DVj1Y6R.png)
- With this access, I can search for the user flag.

  ![Screenshot](https://imgur.com/AA4YMR8.png)

### ⚙️ Privilege Escalation
- To escalate privileges, I uploaded and ran `linpeas.sh` again, this time on the host system.
  ![Screenshot](https://imgur.com/hSkEzxi.png)
- It revealed an interesting file: `/tmp/bash` (owned by `root`)
- I ran it with: `/tmp/bash -p`

  ![Screenshot](https://imgur.com/m3Kbv6G.png)
- And got a root shell. From there, I was able to access the root flag and submit it.

  ![Screenshot](https://imgur.com/Y5cF3pS.png)

