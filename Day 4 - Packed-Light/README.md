# [Writeup] TryHackMe - Hacker Holidays 2026: Day 4 - Packed Light

**Category:** Network Forensics / Cryptography
**Difficulty:** Medium
**Keywords:** Wireshark, Tshark, PCAP Analysis, Covert Channel, Base64, XOR Cipher, Bash Automation, Xxd, Awk

---

## Introduction & Objectives
> *Tiny packets. Odd hours. Suspiciously regular. Someone's smuggling out the data equivalent of a hotel towel every night, folded neatly inside traffic that looks ordinary until you decode it.*

According to the challenge description, we are provided with a network capture file named `capture.pcapng`. Our primary objective is to investigate the traffic dump, uncover a hidden out-of-band data exfiltration channel, reverse the custom encryption, and recover the flag.

### Key Indicators from the Briefing & Stories:
1. Suspicious outbound traffic is routed to a non-standard port: `:8080`.
2. Requests are cyclic and execute **every single second like clockwork** (*"every single second like clockwork"*).
3. There is an architectural anomaly within the header structure (*"request headers are giving not a real app"*).
4. Cryptography is explicitly mentioned as an obstruction (*"what is with the crypto"*).

---

## Step 1: Traffic Filtering & Anomaly Identification in Wireshark
We launch Wireshark and ingest the raw packet capture. To isolate the recurring HTTP web requests over the designated port, we apply the following display filter:

```text
http.request and tcp.port == 8080
```

While auditing the filtered HTTP payload structures, an anomaly stands out inside the request headers: a suspicious parameter inside the cookie field labeled `Cookie: hotel_sess_state=...`.

To visualize the complete timeline of this exfiltration path, we right-click on the `hotel_sess_state` string and select **Apply as Column**. This populates our main packet viewer list with a sequential, chronological stream of short, isolated string tokens. Every token explicitly terminates with a `==` padding character, indicating **Base64** encoding:

* `HA==`
* `AA==`
* `BQ==`
* `Mw==`
* ...

Given that these tiny packets traverse the network strictly 1 second apart, the threat actor is clearly smuggling stolen data **character-by-character**—embedding one obfuscated ASCII byte into each sequential HTTP request cookie.

---

## Step 2: Base64 Decoding & XOR Key Derivation (Console Method)
If we attempt to perform a direct, raw Base64 decoding pass on the very first extracted cookie token (`HA==`):
```bash
echo "HA==" | base64 -d
```
The console returns an empty string or unprintable characters. This occurs because the encoded byte maps to a non-printable ASCII control character. 

To analyze the underlying data block, we pipe the raw decoded stream into the `xxd` hex-dump utility:
```bash
echo "HA==" | base64 -d | xxd
# Output: 00000000: 1c
```

The operation reveals a raw hexadecimal byte values of **`0x1c`**.

### The Cryptanalysis Hypothesis:
We leverage a foundational CTF methodology rule: every flag string on the TryHackMe platform invariably begins with the fixed prefix character **`T`**. Looking at the standard ASCII table, the hexadecimal representation for the uppercase character `T` is **`0x54`**.

Remembering the architectural hint regarding cryptography, we test for a simple **XOR Cipher** mechanism. We exploit the mathematical inversion property of the logical XOR operation (\(A \oplus B = C \implies A \oplus C = B\)). By executing an XOR operation between the captured ciphertext byte and our known expected plaintext character, we can mathematically extract the secret key byte:

```text
0x1c (Ciphertext Byte) ^ 0x54 (Expected Plaintext 'T') = 0x48 (Derived Secret Key)
```

Cross-referencing the resulting hex value **`0x48`** back to the ASCII character chart yields the uppercase character **`H`**. 

The threat actor's pipeline encryption routine is completely reverse-engineered: the malicious script captures a keylogged stroke character, processes it via an **`XOR 0x48`** ('H') mask matrix, and encodes the resulting control block into Base64 for stealth transport.

---

## Step 3: Automation — Single-Command Flag Reassembly via TShark & Bash
Instead of manually processing 30+ network packets or dealing with network transmission distortions (such as TCP Retransmissions that duplicate packets and corrupt the flag layout), we can build an automated, industrial-grade console pipeline using **TShark** and **Bash**:

```bash
tshark -r capture.pcapng -Y "http.request" -T fields -e http.cookie | grep -o "hotel_sess_state=[^;]*" | cut -d'=' -f2 | awk '!x[$0]++' | while read -r token; do hex_val=$(echo -n "$token" | base64 -d 2>/dev/null | xxd -p); if [ ! -z "$hex_val" ]; then printf "\\x$(printf %x $((0x$hex_val ^ 0x48)))"; fi; done; echo ""
```

### Technical Breakdown of the Pipeline:
1. `tshark -r ... -Y "http.request" -T fields -e http.cookie` — Parses the raw `pcapng` capture stream offline, cleanly carving out the textual contents of the `Cookie` strings.
2. `grep` & `cut` — Sanitizes the output buffer, stripping away cookie keys to leave pure raw tokens (`HA==`, `AA==`, etc.).
3. `awk '!x[$0]++'` — **Critical De-duplication Step:** Filters out duplicate sequential packets caused by network latency or TCP retransmissions, ensuring structural integrity of the keystroke order.
4. `while read -r token; do ...` — Instantiates a loop that ingests each unique token, decodes it via `base64`, maps it to a hex byte string via `xxd`, and computes the logical bitwise `XOR 0x48`.
5. `printf "\\x..."` — Programmatically casts the computed integers back into readable ASCII characters on-the-fly, printing the emerging string directly to the terminal output stream.

### Execution Output:
Running the pipeline dynamically decrypts the covert channel and drops the recovered flag parameters sequentially:

```text
THM{***}
```

