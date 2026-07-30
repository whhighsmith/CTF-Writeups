# TryHackMe - Corridor Writeup
**Room:** Corridor  
**Difficulty:** Beginner  
**Tools Used:** Browser Developer Tools, Terminal  
**Platform:** TryHackMe AttackBox  

---

## Introduction

Corridor is a beginner-friendly CTF challenge on TryHackMe centered around **Insecure Direct Object Reference (IDOR)**. The twist in this room is that the URL endpoints are **MD5 hashes** rather than plain numbers — giving the appearance of security through obscurity. The objective was to identify what the hashes represented, generate new ones, and access a resource that shouldn't be publicly available.

---

## Reconnaissance

Upon loading the room, I was presented with a web application where each endpoint in the URL was represented as an MD5 hash, for example:

```
http://<MACHINE_IP>/some_md5_hash_here
```

This immediately suggested an IDOR vulnerability — if the hashes were just MD5 representations of predictable values like numbers, I could generate my own hashes and access unintended resources.

---

## Identifying the Hash Pattern

I copied one of the URL hashes and attempted to identify what value it represented. Rather than using an online tool repeatedly, I used the terminal to generate MD5 hashes for common predictable values:

```bash
for i in {0..10}; do echo -n "$i" | md5sum; done
```

This generated MD5 hashes for the numbers 0 through 10 all at once, which I then compared against the hashes in the URL endpoints to identify the pattern.

The `-n` flag is important here — without it, `echo` adds a newline character which changes the resulting hash entirely.

---

## Exploiting the IDOR

Once I confirmed the hashes were MD5 representations of simple integers, I focused on boundary values that developers commonly overlook. The most promising value to try was **0**, since:

- Most applications start counting from `1` for normal users
- `0` frequently points to an admin account or root resource
- It is often overlooked during development and security testing

I generated the MD5 hash for `0`:
```bash
echo -n "0" | md5sum
```

I then navigated to the corresponding endpoint in the browser:
```
http://<MACHINE_IP>/<md5_of_0>
```

---

## Flag

The endpoint corresponding to the MD5 hash of `0` returned the flag.

---

## Key Takeaways

- **Hashing values does not make them secure.** If the underlying value is predictable (like an integer), hashing it with MD5 provides no real protection — it just adds a layer of obscurity.
- **IDOR vulnerabilities are about missing authorization checks**, not just predictable IDs. The application should verify that the requesting user is authorized to access the requested resource regardless of how the ID is formatted.
- **Always test boundary values** in IDOR challenges — `0`, `-1`, and values outside the normal range are frequently overlooked by developers and are goldmines in CTF challenges and real-world penetration tests.
- **MD5 is not an encryption algorithm** — it is a hashing function and should never be used to protect sensitive identifiers.
- The terminal is faster and more efficient than online hash tools for generating and comparing multiple hashes at once.

---

## Tools Reference

| Tool | Purpose | Command |
|---|---|---|
| Terminal | Generate MD5 hashes for comparison | `echo -n "0" \| md5sum` |
| Terminal | Bulk generate hashes for a range | `for i in {0..10}; do echo -n "$i" \| md5sum; done` |
| Browser | Navigate to hashed endpoints | Manual URL manipulation |

---

## Attack Chain Summary

```
Identified MD5 hashes in URL endpoints
→ Generated MD5 hashes for predictable integers using terminal
→ Matched URL hashes to integer values
→ Tested boundary value 0
→ Navigated to MD5 hash of 0
→ Flag retrieved
```

---

*Writeup by Will | TryHackMe: willyh*
