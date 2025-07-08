# 🔐 HTB Machine: Artificial
![Screenshot](https://imgur.com/ES5mBVz.png)
### 🌐 Machine Information
- **IP:**  10.10.11.74
- **OS:** Linux
### 🧭 Reconnaissance
- I began with a Nmap scan to identify open ports and services: `nmap -A -p- 10.10.11.74 -oN nmap_scan`
![Screenshot](https://imgur.com/ioI2y1Y.png)
- Nmap revealed two open ports—22, which runs SSH, and 80, which runs HTTP.
- Visiting port 80 in the browser redirected me to artificial.htb, so I added it to /etc/hosts: `10.10.11.74 artificial.htb`
![Screenshot](https://imgur.com/3NJy2cU.png)
### 🌐 Exploring the Web Application
- The website has login and registration features. After registering a new account and logging in, I found an option to upload a file—specifically files with the .h5 extension (used for machine learning models).
![Screenshot](https://imgur.com/xo7puoD.png)
- It accepts .h5 files, so I searched for a sample file and uploaded it.
![Screenshot](https://imgur.com/tkr6uBG.png)
- Clicking “View Prediction” simply redirects back to the upload page without any visible result.
### 🧬 Exploiting RCE
- I came across [this article](https://splint.gitbook.io/cyberblog/security-research/tensorflow-remote-code-execution-with-malicious-model) describing an RCE vulnerability using a malicious .h5 file in TensorFlow.
- Based on the information from the website, I used the provided Python code to generate a malicious .h5 file.
![Screenshot](https://imgur.com/7pxEYFn.png)
- I used Python 3.10.16 and TensorFlow version 2.13.1 to run the script, following the recommendation from the artificial.htb website.
- Before uploading, I set up a Netcat listener: `nc -lvnp 4444`
- After uploading test.h5 and clicking "View Prediction," I received a reverse shell—successfully gaining initial access.
![Screenshot](https://imgur.com/axhu3Iq.png)
# 🔍 Initial Foothold and Lateral Movement
- While navigating the file system, I found a .db file inside the instance/ directory. To examine it locally, I started a Python HTTP server on the target machine and downloaded the file to my local system
- Opening it in SQLite revealed some usernames. I also checked /etc/passwd and found a user named gael.
![Screenshot](https://imgur.com/Cr5XRVV.png)
- Using [hashes.com](https://hashes.com/en/decrypt/hash), I cracked the password hash found in the .db file. I then successfully logged in via SSH as gael:
![Screenshot](https://imgur.com/Qj3Jbgg.png)
![Screenshot](https://imgur.com/e4uiESl.png)
- From there, I grabbed the user flag in /home/gael/user.txt.
![Screenshot](https://imgur.com/RwaBl25.png)
### 🧱 Privilege Escalation
- I started by checking `sudo -l` for privilege escalation, but the user couldn't run sudo.
- However, running `ss -tuln` showed a local service running on port 9898.
![Screenshot](https://imgur.com/cQm8S9Q.png)
- To access it, I created an SSH tunnel: `ssh -L 9898:localhost:9898 gael@10.10.11.74`
![Screenshot](https://imgur.com/5cYrekg.png)
- Navigating to http://localhost:9898 revealed a Backrest Web UI. I tried using the same gael credentials but couldn't log in.
![Screenshot](https://imgur.com/wbxicTb.png)
### 📦 Finding Credentials in Backups
- While exploring /var/backups, I discovered a file: backrest_backup.tar.gz.
- I transferred it to my local machine and extracted it.
- After extracting the zip, there will be a backrest directory, and there is a config file in the /backrest/.config directory.
![Screenshot](https://imgur.com/QPuhRk1.png)
- Based on the contents, I assumed it contained the Backrest login credentials and attempted to crack the password using hashes.com.
![Screenshot](https://imgur.com/wVWvKpk.png)
- This revealed another hash that needed to be cracked.
![Screeenshot](https://imgur.com/cJ07KTQ.pngg)
- It was a bcrypt (Blowfish) hash, so I used Hashcat with the rockyou.txt wordlist to try and crack it: `hashcat -m 3200 hash.txt rockyou.txt`
![Screenshot](https://imgur.com/iSIx9n0.png)
- Once cracked, I logged in to Backrest Web using: `backrest_root:!@#$%^`
### 🧨 Gaining Root Access
- Based on this [Backrest article](https://linuxconfig.org/simplify-restic-backups-with-backrest-a-step-by-step-tutorial), I learned that Backrest allows defining shell commands during repository creation.
- I used this to craft a malicious repository that triggers a reverse shell as root.
![Screenshot](https://imgur.com/AzGCHJT.png)
- After submitting the repository, I went back to my terminal and started a Netcat listener on port 4444.
- I went back to the Backrest interface, opened the repository I had just created, and used the “Check Now” feature to see if the script executed.
![Screenshot](https://imgur.com/VsXKHW6.png)
- And it worked — I got a shell as root!
- With root access, I grabbed the root flag from /root/root.txt.
![Screenshot](https://imgur.com/CqXLLXO.png)