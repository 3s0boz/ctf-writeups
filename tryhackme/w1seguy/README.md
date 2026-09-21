# W1seGuy - TryHackMe

Linux target running a custom Python service on TCP 1337. The service hands out a flag
encrypted with a repeating-key XOR and only releases a second flag to whoever can prove
they recovered the key. There is no shell and no privilege escalation: the whole room is
a known-plaintext attack against a cipher that reuses its key.

Target: `10.65.144.250`

---

## Reconnaissance

The service listens on TCP 1337. Connecting to it with netcat is enough to get the first
half of the challenge:

```bash
nc 10.65.144.250 1337
```

The server returns the first flag as a hex string and then asks for the encryption key:

```
This XOR encoded text has flag 1: <hex string>
What is the encryption key?
```

---

## Source Code Analysis

The room publishes the source of the service, which removes any guesswork about the
cipher. Two parts matter.

The key is five characters drawn from letters and digits:

```python
res = ''.join(random.choices(string.ascii_letters + string.digits, k=5))
key = str(res)
```

That is 62^5, roughly 916 million combinations. The number looks protective and is not,
for the reason the next block makes clear:

```python
for i in range(0, len(flag)):
    xored += chr(ord(flag[i]) ^ ord(key[i % len(key)]))
hex_encoded = xored.encode().hex()
```

The plaintext is XORed byte by byte against the key, and `i % len(key)` wraps the key
around every five bytes. This is a repeating-key XOR, not a one-time pad. A one-time pad
is unbreakable because the key is as long as the message and never reused. Here the key
is five bytes long and reused for the entire flag.

One detail worth noting in the published source: `setup()` takes `server` as a parameter
and never uses it, and it redefines `flag` locally with a placeholder value that shadows
the real flag read from `flag.txt` in the global scope. The second flag, the one handed
out after the key check, is still the real one.

---

## Exploitation - Known-Plaintext Attack

XOR is symmetric, which is the property that breaks this scheme:

```
cipher = plain XOR key    ->    key = cipher XOR plain
```

Any byte of plaintext that can be guessed reveals the corresponding byte of the key. THM
flags always start with `THM{`, so the first four bytes of the plaintext are known, and
they hand over four of the five key bytes directly:

```python
cipher = bytes.fromhex(hex_encoded)
known = "THM{"
key = [chr(cipher[i] ^ ord(known[i])) for i in range(4)]
```

Only the fifth byte is left. The flag also ends with `}`, so its position in the key
cycle can be computed with `(len(cipher) - 1) % 5`. The simpler route is to try all 62
candidates and keep the one that decrypts to a fully printable string in the expected
format:

```python
import string

for c in string.ascii_letters + string.digits:
    candidate = key[:4] + [c]
    plain = ''.join(chr(cipher[i] ^ ord(candidate[i % 5])) for i in range(len(cipher)))
    if plain.startswith("THM{") and plain.endswith("}") and plain.isprintable():
        print(candidate, plain)
```

That prints the key and the decrypted first flag.

---

## Second Flag

The service compares the submitted key against the one it generated. Sending the
recovered key back at the `What is the encryption key?` prompt returns the second flag,
read from `flag.txt` on the target.

---

## Key Takeaways

- Repeating-key XOR with any known plaintext does not need brute forcing. Rearranging
  `cipher = plain XOR key` into `key = cipher XOR plain` recovers the key directly, one
  byte of key per byte of guessed plaintext.
- A fixed flag format is known plaintext. `THM{` at the start and `}` at the end cover a
  five-byte key almost entirely on their own.
- Key space size is not strength. 62^5 is irrelevant once the key is shorter than the
  known plaintext and gets reused cyclically.
- Repeating-key XOR shows up in the field as obfuscation rather than protection: strings
  and payloads inside malware samples, C2 configuration, shellcode encoders. The approach
  is the same there, look for a known header or recurring string and derive the key from
  it instead of attacking the key space.

---

## Disclaimer

This lab was completed in a controlled environment provided by TryHackMe. All actions
were performed strictly for educational purposes.
