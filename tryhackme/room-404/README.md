# Room 404 - TryHackMe

Web target running on port 8080 hosting "Byte Lotus - Guest Experience Platform". Directory listing is enabled and the application's `.git` directory is exposed on the webroot. The entire source repository is recoverable without authentication, and the flag sits in plaintext inside a committed `README.md`. No shell, no privilege escalation - this is a pure enumeration room, and the point of it is the recovery workflow rather than the capture.

---

## Enumeration

### Web Service

Target: `10.64.157.196:8080`. Directory listing enabled across the webroot, including `/.git/`.

Browsing `/.git/objects/` returned a live index:

```
0a/  0f/  25/  a5/  fa/  info/  pack/
```

Five loose object prefixes. For a repository with a single initial commit the minimum object set is one commit, one tree, and one blob per file - five objects therefore implies a very small tree, three or four files at most.

### Reflog

`.git/logs/HEAD` is a plaintext file and is the fastest first read:

```
0000000000000000000000000000000000000000 0f13550b4cb13e9f30c61d5b342c532d21e45bda night-shift <dev@byte-lotus.internal> 1762049640 +0000	commit (initial): initial Byte Lotus guest platform
```

Field layout is `<old-sha> <new-sha> <committer name> <email> <epoch> <tz>` followed by a tab and `<action>: <message>`.

What this yields before a single object is decoded:

- An all-zero left SHA marks the initial commit - there is no prior history to hunt for.
- One recorded HEAD movement only, so no later commit removed a secret.
- Username candidates `night-shift` (git committer name) and `dev` (email localpart).
- Internal domain `byte-lotus.internal`.
- Commit timestamp `1762049640` = 2025-11-02 02:14 UTC.

`.git/config` and `.git/info/exclude` were also retrievable but carried nothing: both are stock templates. The only signal was by omission - no `[remote]` section, so the repository was created with `git init` on the host rather than cloned, which rules out credentials embedded in a remote URL.

---

## Repository Recovery

With directory listing enabled a recursive `wget` is sufficient - no dedicated tooling required:

```bash
mkdir room404 && cd room404
wget -r -np -nH -R "index.html*" http://10.64.157.196:8080/.git/
```

- `-r` recursive
- `-np` do not ascend above `/.git/`
- `-nH` do not create a host-named directory
- `-R "index.html*"` discard the generated listing pages

Result: 46 files, 32K, all five loose objects plus `refs/heads/main` and `logs/refs/heads/main`.

When directory listing is **disabled** this approach fails, because `wget` has no index to walk. That is the case `git-dumper` exists for: it starts from `.git/HEAD`, follows refs, and fetches objects by SHA without needing a listing, packfiles included.

### Object Map

One command returns the entire repository structure:

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

One commit, one tree, three blobs. No packfile, no unreachable objects - the dump is complete. `git fsck --full` confirms integrity; `git reset --hard HEAD` restores the working tree.

### Tree

```bash
git cat-file -p fa45dbd69394ea9e13683d9efb6a0220daac59d4
```

```
100644 blob a5965c580fee91d852e5b19a8290da02d2926523	README.md
100644 blob 2575ab073f67615a27135663ed36794c2d2584fb	app.js
100644 blob 0a12caa4e52a965e89e5eccf5760924b21aacbf7	index.html
```

Three static files. No Python, no `requirements.txt`, no server-side framework in the repository.

---

## Findings

`README.md` is an internal staging document. It names the repository as the guest application plus a "concierge personalization service", warns against deploying the folder to production, and carries the flag in plaintext under a line reading `Staging flag (remove before launch)`.

`app.js` is a front-end stub. It declares an API base path:

```javascript
const API = "/api/guest";
```

The comment describes personalization as served from a separate profiling service, which points at a second component not present in this repository.

---

## Dead End

Before recovering the repository, `exploit/multi/http/werkzeug_debug_rce` was considered - selected from the room's framing rather than from evidence. There was no Werkzeug banner, no interactive debugger traceback, and no dependency manifest suggesting Flask. The recovered tree confirmed the absence outright.

The ordering that avoids this: confirm the service, confirm the version, confirm the vulnerable condition is present on this target, and only then select the tool. Choosing a module first and firing it to find out burns time and produces "it didn't work" without producing knowledge.

---

## Key Takeaways

- An exposed `.git` directory leaks the full source plus complete commit history. The flag is the least valuable thing in it - removed credentials, API keys, internal paths and real usernames survive in the object store and are recoverable with `git fsck --unreachable` even when no longer referenced.
- With directory listing enabled, `wget -r -np -nH -R "index.html*"` is the whole dump. `git-dumper` is the tool for the far more common case where listing is off.
- Do not read `.git` file by file. `git cat-file --batch-all-objects --batch-check` maps every object in one command, and `git reset --hard HEAD` reconstructs the working tree. `config` and `info/exclude` are stock templates in every repository in existence and almost never carry signal.
- The reflog is an identity source before it is a history source. Committer name, email localpart and internal domain are username and vhost candidates harvested without decoding a single object.
- The repository tells you which stack is running, and therefore which exploits are even applicable. Three static files with no Flask closed a line of enquiry that a room name had opened.

---

## Disclaimer

This lab was completed in a controlled environment provided by TryHackMe. All actions were performed strictly for educational purposes.
