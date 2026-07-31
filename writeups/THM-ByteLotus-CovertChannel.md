# TryHackMe - Covert Communication Channel Writeup
**Event:** Byte Lotus  
**Difficulty:** Intermediate  
**Tools Used:** Wireshark, PyCharm, Python (pyshark, base64)  
**Platform:** Personal Machine + TryHackMe  

---

## Introduction

This challenge involved analyzing a network packet capture (`suspicious_http.pcapng`) to identify a covert communication channel being used to exfiltrate data. The objective was to:

1. Identify where the exfiltrated data was being hidden
2. Reassemble and decode the data
3. Retrieve the flag from the reconstructed output

The core techniques involved **network forensics**, **malware code analysis**, **XOR decryption**, and **custom Python scripting**.

---

## Reconnaissance — Protocol Analysis

I opened the capture file in **Wireshark** and ran an initial protocol analysis via **Statistics → Protocol Hierarchy** to get an overview of the traffic.

**Key findings:**
- No ICMP traffic — ruling out ICMP tunneling
- **18 DNS packets** — low volume, less suspicious
- **62 HTTP packets** including **30 packets of Line-based text data carrying 154KB** — highly suspicious for unencrypted HTTP
- **222 TLS packets** and **179 QUIC packets** — encrypted, harder to inspect initially

The volume of plain text data being transferred over unencrypted HTTP was immediately suspicious and became the primary focus.

---

## Identifying the Primary Conversation

Via **Statistics → Conversations**, I identified the primary conversation of interest:

| Address A | Address B | Packets | Bytes |
|---|---|---|---|
| c0:56:27:cc:a7:9e | e8:9c:25:df:b2:51 | 1,144 | 489 KB |

This conversation accounted for the vast majority of traffic and was clearly the focus of the investigation.

---

## Discovering the Malicious Script

I filtered for HTTP traffic:
```
http
```

I identified an HTTP packet with a `Content-Type` of `text/x-python` — a non-standard content type that immediately raised a red flag. Right clicking the packet and selecting **Follow → HTTP Stream** revealed the full request and response.

The server at `byte-lotus-hotel.thm:8080` had served a Python script located at `/temp/updates.py`.

---

## Malware Analysis

The recovered Python script was a **keylogger** disguised as a legitimate update service. Analysis of the code revealed the following behavior:

**1. Keystroke Capture**
The script used the `pynput` library to capture every keystroke typed by the victim.

**2. XOR Encryption**
Each captured keystroke was encrypted using XOR with a hardcoded key constructed from two strings:
```python
p1 = "H0t3lSt@ff0Nly"
p2 = "K3epS3cr3t!"
```
Combined XOR key: `H0t3lSt@ff0NlyK3epS3cr3t!`

**3. Base64 Encoding**
The encrypted keystroke was then Base64 encoded to make it appear as a legitimate session token.

**4. Covert Exfiltration via Cookie Header**
Each encoded keystroke was exfiltrated by embedding it in the `Cookie` header of HTTP GET requests sent to the C2 server:
```
Cookie: hotel_sess_state=<base64_encoded_encrypted_character>
```

This technique is particularly stealthy because:
- Cookie headers are common in normal web traffic and don't raise immediate suspicion
- Each request only carries one character, making the data difficult to spot manually
- The data appears to be a legitimate session token

---

## Extracting the Exfiltrated Data

I filtered Wireshark for the exfiltration traffic:
```
http.cookie contains "hotel_sess_state"
```

This revealed a large stream of HTTP GET requests, each containing one encrypted and encoded keystroke in the cookie header.

---

## Decryption Script

I wrote a custom Python script to automatically extract all `hotel_sess_state` cookie values from the pcap file, Base64 decode them, and XOR decrypt them using the recovered key:

```python
import base64
import pyshark

XOR_KEY = "H0t3lSt@ff0NlyK3epS3cr3t!"

def xor_decrypt(data: bytes, key: bytes) -> bytes:
    return bytes(b ^ key[i % len(key)] for i, b in enumerate(data))

def decode_keystroke(b64_string: str) -> str:
    try:
        encrypted = base64.b64decode(b64_string)
        decrypted = xor_decrypt(encrypted, XOR_KEY.encode('utf-8'))
        return decrypted.decode('utf-8')
    except Exception as e:
        return f"[ERROR: {e}]"

cap = pyshark.FileCapture(
    "suspicious_http.pcapng",
    display_filter='http.cookie contains "hotel_sess_state"'
)

keystrokes = []
for packet in cap:
    try:
        cookie = packet.http.cookie
        for part in cookie.split(';'):
            part = part.strip()
            if part.startswith('hotel_sess_state='):
                b64_value = part.split('=', 1)[1]
                keystrokes.append(decode_keystroke(b64_value))
    except AttributeError:
        continue

cap.close()
print(''.join(keystrokes))
```

**Note:** When specifying file paths in Python on Windows, use a raw string to avoid Unicode escape errors:
```python
pcap_file = r"C:\Users\...\suspicious_http.pcapng"
```

---

## Flag

Running the decryption script successfully reassembled and decrypted all captured keystrokes, revealing the flag.

---

## Key Takeaways

- **Covert channels don't have to be exotic** — hiding data in standard HTTP Cookie headers is a simple but effective exfiltration technique that can blend into normal traffic
- **Protocol Hierarchy in Wireshark** is an invaluable first step in any pcap analysis — it immediately surfaces anomalies without having to scroll through every packet
- **Non-standard Content-Types** like `text/x-python` being served over HTTP are a strong indicator of malicious activity
- **XOR encryption with a hardcoded key** is a common technique in malware — once the key is recovered from the source code the encryption is trivially reversible
- **Custom decryption scripts** are a powerful tool in forensic investigations — understanding how the malware works allows you to build the exact tool needed to reverse it
- **C2 communication over HTTP** is common in real world malware because it blends in with normal web traffic and is often allowed through firewalls

---

## Tools Reference

| Tool | Purpose |
|---|---|
| Wireshark | Packet capture analysis, protocol hierarchy, HTTP stream following |
| Python (pyshark) | Automated extraction of cookie values from pcap |
| Python (base64) | Decoding Base64 encoded keystrokes |
| PyCharm | Python script development and execution |

---

## Attack Chain Summary

```
Opened pcap in Wireshark
→ Protocol Hierarchy revealed suspicious HTTP plain text traffic
→ Followed HTTP stream → discovered malicious Python keylogger script
→ Analyzed keylogger → identified XOR key and Cookie exfiltration method
→ Filtered for hotel_sess_state cookie headers
→ Wrote Python script to extract, decode, and decrypt keystrokes
→ Reassembled keystrokes → flag retrieved
```

---

*Writeup by Will | TryHackMe: willyh*
