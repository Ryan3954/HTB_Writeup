# 🧬 HTB Machine: Code
![Screenshot](https://imgur.com/tIcikoY.png)
### 🌐 Machine Information
- **IP:**  10.10.11.58
- **OS:** Linux
### 🧭 Information Gathering
- I started with an Nmap scan to identify open ports and running services: `nmap -A -p- 10.10.11.62 -oN nmap_scan`
![Screenshot](https://imgur.com/XqXmywU.png)
- Nmap Results 2 open ports: 22 (ssh), 5000 (HTTP Python-based web service)
- Visiting port 5000 in a browser revealed a web-based Python code editor. However, it included multiple restrictions
![Screenshot](https://imgur.com/0eQ4B7f.png)
- Common functions and modules like `eval, exec, open, read, import, os, and __builtins__` were blocked.
- This indicated a Python sandbox or "Python Jail."

### 🧨 Escaping the Python Jail
- After researching common Python Jail bypass techniques, Here's the payload I used to run the whoami command
![Screenshot](https://imgur.com/c88s96E.png)
- This returns the current user.

    ![Screenshot](https://imgur.com/q3S22Sc.png)
    
### 📂 Exploring the Filesystem
- Using similar payloads, I navigated the filesystem and confirmed the working directory:
![Screenshot](https://imgur.com/Ta8GRLj.png)
![Screenshot](https://imgur.com/4gMLipz.png)
- After exploring the `app-production` directory, I found the user flag at `/home/app-production/user.txt`
- So i used this payload to retrieve the user flag
![Screenshot](https://imgur.com/425Bieg.png)
- While exploring the `app-production` directory earlier, I found another interesting file named `database.db` inside `instance` directory.
- So I downloaded the file to my local machine with this payload:
![Screenshot](https://imgur.com/Op8pk9J.png)
- Before running the payload on the website, I need to use the command: `nc -lvnp 5555 > code_database.db`
- After getting the file, I opened `code_database.db` with `sqlite3`
![Screenshot](https://imgur.com/px65BXU.png)
- Inside the database, I found credentials for a user named martin. The password was hashed, so I used hashes.com to crack it.
![Screenshot](https://imgur.com/8JDFRcQ.png)

### 🔐 Gaining SSH Access
- With the recovered credentials, I logged into the machine via SSH: `ssh martin@10.10.11.62`
![Screenshot](https://imgur.com/tTyG86X.png)
- Successfully logged in as martin.

### 🔼 Privilege Escalation
- I checked martin’s sudo permissions: `sudo -l`
![Screenshot](https://imgur.com/OunL0c5.png)
- Output showed martin can run `backy.sh` as root without a password.
- `backy.sh` appears to read from a JSON file and archives a specified directory provided by the JSON file.
![Screenshot](https://imgur.com/b20PNv0.png)

### 📦 Abusing `backy.sh`
- The script only allows archiving directories under /var/ or /home/.
- Attempting to archive /root directly failed. So, I tried using traversal bypass.
![Screeshot](https://imgur.com/bRpLYWB.png)
- But it doesn't works
![Screenshot](https://imgur.com/IaGQf6I.png)
- Targeted Directory became /home/martin/root, because the script sanitize `../` by replacing it with `""`
![Screenshot](https://imgur.com/d1IC0Jz.png)
- So I used another payload:
![Screenshot](https://imgur.com/rNw3KCx.png)
- It Works!
- After archiving the `root` directory, I sent it to my local machine
- With that, I opened the archive and extract the `root flag`
![Screenshot](https://imgur.com/rbsNLXT.png)