# Hack The Box: Ignition — Machine Writeup

### 📌 Overview & Metadata
*   **Target:** Ignition
*   **Concepts:** Directory Enumeration, Brute-Force Authentication, Weak Credentials
*   **Pwn Date:** 18 Aug 2026

### 🧠 Tool Breakdown: Gobuster
Gobuster is a fast, highly efficient command-line tool written in Go, primarily used to brute-force URIs (directories and files) inside web servers and DNS subdomains. It is an essential asset for mapping out a web application's hidden attack surface.

For this engagement, the following command was executed:
`gobuster dir -u http://ignition.htb -w admins.txt`

**Flag Explanations:**
*   `dir`: This is the mode specification, telling Gobuster to perform standard directory and file enumeration.
*   `-u`: This flag specifies the target URL, which in this case was `http://ignition.htb`.
*   `-w`: This flag defines the path to the custom wordlist used for fuzzing, which was `admins.txt`.

### 🔍 Reconnaissance & Enumeration
By executing the Gobuster directory brute-force attack, I successfully uncovered a hidden login panel at the `/admin` endpoint, which returned a 200 OK HTTP status code.

### 🎯 Exploitation: Burp Suite Intruder
With the login portal identified, I captured an authentication attempt using Burp Suite's Proxy and forwarded the POST request to the Intruder tool[cite: 38]. 

*   **Attack Type:** I configured a "Sniper" attack. This attack type takes a single set of payloads and inserts them one by one into the specified payload position.
*   **Payload Position:** I set a single payload marker (`§`) exclusively around the `password` parameter's value (`login%5Bpassword%5D=§keylogger§`) to aggressively fuzz the password field while keeping the `admin` username static. 

### 🚩 The Breach
I initiated the brute-force attack against the endpoint. Payload #46, which tested the string `qwerty123`, returned a 302 Found HTTP status code alongside a `Set-Cookie: admin=...` header. This redirect confirmed a successful authentication bypass, granting full administrative access and yielding the final flag.

### 💡 Remediation & Lessons Learned
The compromise of this machine underscores a fundamental security principle:
*   **Weak Credentials:** We should never leave default, predictable, or weak passwords (like `qwerty123`) on administrative endpoints. Passwords must enforce strict complexity requirements.
*   **Lack of Rate Limiting:** The application allowed an aggressive, automated Burp Suite Intruder attack without blocking the IP or locking the account. Implementing strict rate limiting and account lockout policies is mandatory to prevent brute-forcing.
