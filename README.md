# 🏝️ TryHackMe - Hacker Holidays 2026: Full Walkthroughs (Days 1–8)

Welcome to my repository containing technical notes, exploit steps, and detailed writeups for the **TryHackMe Hacker Holidays 2026** event. 

This repository documents my journey of compromising the infrastructure of the fictional **Byte Lotus Resort**, moving from initial artificial intelligence manipulation to reverse shells and cloud database dumps.

---

## 📊 Challenge Progress Tracker (Days 1–8)

| Day | Challenge Name | Category | Difficulty | Vulnerability / Core Technique | Status | Walkthrough |
|:---:|:---|:---|:---:|:---|:---:|:---:|
| **01** | **The Concierge Knows Too Much** | AI Security | Easy | Identity Impersonation & Prompt Injection | 🟢 Solved | [View](https://github.com/aliasdef/THM-Hacker-Holidays-2026-Writeups/tree/main/Day-01-The-Concierge) |
| **02** | **Room 404** | Web Exploitation | Easy | Exposed Git Repository (`/.git/` source disclosure) | 🟢 Solved | [View](https://github.com/aliasdef/THM-Hacker-Holidays-2026-Writeups/tree/main/Day-02-Room-404) |
| **03** | **Complimentary** | Cloud Security | Medium | AWS IAM Over-privileged Guest Access & DB Dump | 🟢 Solved | [View](https://github.com/aliasdef/THM-Hacker-Holidays-2026-Writeups/blob/main/Day%203%20-%20Complimentary/README.md) |
| **04** | **Packed Light** | Network Forensics | Medium | Covert Channel & Reassembling XOR-encoded Cookies | 🟢 Solved | [View](https://github.com/aliasdef/THM-Hacker-Holidays-2026-Writeups/tree/main/Day%204%20-%20Packed-Light) |
| **05** | **Beach Bar** | Boot2Root | Easy | Jukebox Input Arbitrary Command Injection & RCE | 🟢 Solved | [View](https://github.com/aliasdef/THM-Hacker-Holidays-2026-Writeups/blob/main/Day%205%20%20-%20Beach%20Bar/README.md) |
| **06** | **Overheard at Breakfast** | OSINT | Easy | Email Hashing Reconnaissance via Gravatar Profiles | 🟢 Solved | [View](./Day-06-Overheard/) |
| **07** | **Do Not Disturb** | Boot2Root / Web | Medium | NoSQLi Auth Bypass ➡️ EJS SSTI ➡️ Disk Group Abuse | 🟢 Solved | [View](./Day-07-Do-Not-Disturb/) |
| **08** | **Towel on the Sunbed** | Web / Logic | Medium | Race Condition (Burp Single-Packet Attack on Timer) | 🟢 Solved | [View](./Day-08-Towel-Sunbed/) |

---

## 🛠️ Summary of Exploitation Vectors Covered

### 1. Web & API Vulnerabilities
* **NoSQL Injection:** Injecting JSON objects (`$ne` query operator) to bypass MongoDB authentication logic.
* **Race Condition (TOCTOU):** Executing HTTP parallel requests via Burp Suite (Single-Packet Attack) to bypass 24-hour reward window limits.
* **Server-Side Template Injection (SSTI):** Code execution using EJS (Embedded JavaScript) backend templating blocks.
* **Command Injection:** Forcing application backends to route unsanitized terminal commands.

### 2. Infrastructure & Cloud Security
* **Source Code Disclosure:** Extracting `.git` tracking folders manually to rebuild staging code repositories offline.
* **AWS Cloud Misconfigurations:** Leveraging leaked unauthenticated AWS programmatic identity IDs to map out and dump database segments.
* **Network Traffic Forensics:** Utilizing Wireshark to filter malicious traffic, reassemble scattered HTTP chunk data, and reverse XOR masking arrays.

### 3. Linux Privilege Escalation
* **Pivoting:** Accessing Node.js active debugger instances (`node inspect 127.0.0.1:9229`) internally from local shell environments.
* **Abusing Group Memberships:** Leveraging the `disk` group layer context to mount or interact with direct block stores (`/dev/nvme0n1p1`) via `debugfs` to bypass root access checks.

---

## 🔗 Connected Profiles
* **Medium Blog:** [https://medium.com/@hamemeee]
* **TryHackMe Profile:** [https://tryhackme.com/p/hamemeee]

*Feel free to star ⭐️ this repository if you find these references helpful as you track your own Hacker Holidays solutions!*
