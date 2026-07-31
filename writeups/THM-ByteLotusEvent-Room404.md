# TryHackMe - Room 404 Writeup
**Event:** Byte Lotus

**Room:** Room 404

**Difficulty:** Beginner

**Tools Used:** Gobuster, git-dumper

**Platform:** TryHackMe AttackBox 

---
# Introduction:
Room 404 is a beginner-friendly CTF challenge from the TryHackMe Byte Lotus event. The objective was to find a hidden flag on a web server running on port 8080. The core technique used was directory enumeration followed by source code dumping.

---
# Reconnaissance
Upon starting the machine, I was given a target IP address with port 8080 open. I visited the web server in the browser to get an initial look at the application:
```
http://<MACHINE\_IP>:8080
```
Nothing immediately stood out on the surface, so I moved on to directory enumeration to find hidden paths.

---
# Directory Enumeration

I used Gobuster with the `common.txt` wordlist to enumerate hidden directories and files on the web server:
```bash
gobuster dir -u http://<MACHINE\_IP>:8080 -w /usr/share/wordlists/dirb/common.txt
```
Key finding:
```
/.git/HEAD
```
Gobuster revealed an exposed `.git` directory — a significant misconfiguration that allows anyone to access the server's entire Git repository, including source code, commit history, and potentially sensitive files.
---
Source Code Dumping
With the `.git` directory exposed, I used git-dumper to pull the entire repository down to my local machine:
```bash
git-dumper http://<MACHINE\_IP>:8080/.git/ ./output
```
This dumped all repository files into an `./output` folder. I navigated into the directory and listed all files including hidden ones:
```bash
cd output
ls -la
```
Files found:
`app.js`
`index.html`
`README.md`
---
Analysis
I examined each file for useful information.
app.js revealed a hidden API endpoint:
```
/api/guest
```
I attempted to access this and several other common API endpoints (`/api/admin`, `/api/user`, `/api/login`) in the browser, but all returned 404 errors.
---
Flag:

The flag was found inside the `README` file in the dumped repository.

---
# Key Takeaways
Exposed `.git` directories are a serious web vulnerability. When a `.git` folder is publicly accessible, attackers can dump the entire source code, commit history, and any sensitive data ever committed to the repository.
Directory enumeration is often the first step in uncovering misconfigurations that aren't visible on the surface.
Always check README and documentation files — developers frequently leave sensitive notes or information in plain text files without considering the security implications.
Commit history should always be reviewed during source code analysis, as credentials or sensitive data committed in the past remain accessible even if later deleted.

---
Tools Reference
Tool	Purpose	Command
Gobuster	Directory enumeration	`gobuster dir -u <URL> -w <wordlist>`
git-dumper	Dump exposed git repository	`git-dumper <URL>/.git/ ./output`
---
Writeup by Will | TryHackMe: willyh
