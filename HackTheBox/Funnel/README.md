# Hack The Box: Funnel — Machine Writeup

### 📌 Overview & Metadata
*   **Target:** Funnel (Linux)
*   **Concepts:** Information Leakage, SSH Tunneling, Local Port Forwarding, Database Enumeration
*   **Pwn Date:** 14 Aug 2026

### 🧠 Core Concepts: PostgreSQL & The Loopback Interface

Before diving into the exploit, it is vital to understand the target service and why it was restricted in the first place.

**What is PostgreSQL?**
PostgreSQL (often called Postgres) is a highly advanced, open-source object-relational database management system. It is heavily relied upon in enterprise environments to manage and store backend data for web applications. By default, it operates on TCP port 5432.

**What is the Loopback (`lo`) Address?**
The IP address `127.0.0.1` is known as the "loopback" address, attached to a virtual network interface called `lo`. When a computer sends network traffic to this address, it never touches the physical network; instead, it "loops back" directly into the computer's own network stack. 

**Why Bind Services to the Loopback Address?**
This is a standard security hardening practice. By configuring a sensitive internal service (like a PostgreSQL database) to listen *only* on `127.0.0.1`, systems administrators ensure it cannot be accessed directly from the external internet or the broader local network. Only processes running on that specific server can interact with the database, dramatically reducing the attack surface.

### 🔍 Reconnaissance & Initial Access

The engagement began with an Nmap scan, which revealed an open FTP port (21) allowing anonymous login. Exploring the FTP directory yielded two critical files: a `password_policy.pdf` and a welcome email named `welcome_28112022`. 

Analyzing the leaked documents provided the keys to the castle. The welcome email exposed a valid username (`christine`), while the password policy PDF explicitly stated the company's default password (`funnel123#!#`). 

I combined these leaked credentials to successfully secure an SSH foothold as the user `christine`.

### 🚇 Evasion: SSH Tunneling & Port Forwarding

Once inside, I needed to map the internal network surface. Running `ss -tln` to check listening ports revealed port 5432 (PostgreSQL) listening internally on `127.0.0.1:5432`.

Because the database was bound exclusively to the loopback address, any direct connection attempts from my Kali machine would be dropped. To bypass this internal firewall restriction, I utilized **Local Port Forwarding**—a technique that redirects network traffic from a local port, through a secure SSH tunnel, to a remote destination.

I executed the following command:

```bash
ssh -L 1234:127.0.0.1:5432 christine@10.129.228.195
This created an encrypted tunnel, mapping the remote, isolated PostgreSQL database directly to port 1234 on my local Kali machine.

🎯 Data Extraction & Flag Capture
With the tunnel established, I could now interact with the database as if it were running on my own machine. I used the psql command-line utility to connect to the forwarded port using Christine's credentials.

Bash
psql -h 127.0.0.1 -p 1234 -U christine
Once inside the PostgreSQL prompt, I proceeded with standard database enumeration:

List Databases: Used \l to identify available databases, discovering one named secrets.

Connect to Database: Used \c secrets to switch my context to the target database.

List Tables: Used \dt to list the relations, revealing a table simply named flag.

Extract Data: I queried the table using standard SQL syntax to capture the final objective:

SQL
SELECT * FROM flag;
Flag: cf277664b1771217d7006acdea006db1

💡 Remediation & Lessons Learned
This challenge is a masterclass in how attackers chain minor flaws into total compromise:

Information Disclosure: Anonymous FTP should never contain sensitive policy documents or internal communications.

Default Credentials: Passwords like funnel123#!# must be forced to reset upon first login.

Defense-in-Depth Failure: While binding the database to localhost was a good security measure, it was entirely negated by the compromised SSH account, demonstrating that internal services are only as secure as the perimeter protecting them.
