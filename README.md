[README.md](https://github.com/user-attachments/files/29437519/README.md)
# WestWild CTF — Walkthrough

A walkthrough of **WestWild**, a boot2root virtual machine used for practicing enumeration and exploitation techniques in an isolated lab environment. The exercise covers network reconnaissance, SMB share enumeration, anonymous file access, SSH foothold acquisition, and post-exploitation flag discovery.

> 
## Summary

| | |
|---|---|
| **Target** | WestWild (boot2root VM) |
| **Target IP** | `192.168.190.131` |
| **Attacker** | Kali Linux 2026.1 (VMware Workstation) |
| **Network** | Host-only / isolated lab segment |

## Table of Contents

- [Objective](#objective)
- [Reconnaissance](#reconnaissance)
- [Initial Access via SMB](#initial-access-via-smb)
- [Post-Exploitation: Script Analysis & FLAG2](#post-exploitation-script-analysis--flag2)
- [Flags Captured](#flags-captured)
- [Attack Path Summary](#attack-path-summary)
- [Lessons & Hardening Notes](#lessons--hardening-notes)

## Objective

- Identify open services and their versions through active network scanning.
- Enumerate Samba/NetBIOS shares for anonymously accessible files.
- Recover the first flag from an exposed SMB share.
- Establish an interactive SSH session using credentials discovered during enumeration.
- Locate and analyze a suspicious script on the target, then recover the second flag.

## Reconnaissance

### Port Scanning (Nmap)

```bash
nmap -A 192.168.190.131
```

![Nmap scan results](images/01-nmap-scan.jpg)
*Nmap scan showing SSH, HTTP, and Samba services on the WestWild target.*

Key findings:

- **22/tcp** — OpenSSH 6.6.1p1 (Ubuntu 2ubuntu2.13)
- **80/tcp** — Apache httpd 2.4.7 (Ubuntu), no page title set
- **139/445/tcp** — Samba smbd (3.X–4.X / 4.3.11-Ubuntu), workgroup `WORKGROUP`
- **NetBIOS/SMB** — computer name `westwild`, guest access supported, message signing **disabled** (flagged by nmap as "dangerous, but default")

The combination of guest-accessible SMB shares and disabled message signing made Samba the most promising initial enumeration vector.

### Samba/NetBIOS Enumeration (enum4linux)

```bash
enum4linux
```

![enum4linux usage](images/02-enum4linux.jpg)
*enum4linux usage/help output, used to plan share and user enumeration against the target.*

## Initial Access via SMB

### Anonymous SMB Login & FLAG1

With guest SMB access available, `smbclient` was used to connect directly to a share named `wave` without credentials:

```bash
smbclient //192.168.190.131/wave
```

Anonymous login succeeded immediately, exposing two files:

- `FLAG1.txt` (93 bytes)
- `message_from_aveng.txt` (115 bytes)

```bash
smb: \> get FLAG1.txt
cat FLAG1.txt
```

![SMB access, FLAG1, and SSH session](images/03-smb-flag1-ssh.jpg)
*Anonymous SMB access to the "wave" share, FLAG1 retrieval, and the resulting SSH session as user wavex.*

`message_from_aveng.txt` was reviewed for context but is a narrative element rather than a credential.

### SSH Foothold

`FLAG1.txt` is Base64-encoded. Decoding it reveals a username and password hint:

```text
Flag1{Welcome_T0_THE-W3ST-W1LD-B0rder}
user:wavex
password?pen
```

Using this, an SSH session was opened:

```bash
ssh wavex@192.168.190.131
```

After accepting the host's ED25519 key fingerprint, authentication succeeded, dropping into an interactive shell on Ubuntu 14.04.6 LTS as `wavex`. A writable-directory sweep was run next:

```bash
find / -writable-type d 2>/dev/null
cd /usr/share/av/westsidesecret
ls
```

This revealed a suspicious script, `ififoregt.sh`, sitting inside a fake "antivirus" path (`/usr/share/av/westsidesecret`) — an unusual location placed there as part of the challenge rather than genuine AV software.

## Post-Exploitation: Script Analysis & FLAG2

```bash
cd /usr/share/av/westsidesecret
vim ififoregt.sh
rm ififoregt.sh
vim ififoregt.sh
cd
vim FLAG2.txt
cat FLAG2.txt
```

![Reviewing the planted script and recovering FLAG2](images/04-flag2-cleanup.jpg)
*Reviewing the planted script in /usr/share/av/westsidesecret, recovering FLAG2, and cleanup of web/SMB services.*

`FLAG2.txt` contained a themed greeting string along with an in-challenge message prompting players to screenshot and share their result — typical "proof of completion" flavor text from the CTF author, noted here for completeness only.

Following flag capture, the session also touched `/var/www/html` before winding down by stopping `smbd`/`nmbd` and closing the SSH connection — standard cleanup, not part of the core exploit path.

## Flags Captured

| Flag | Source / Method | Value |
|------|------------------|-------|
| FLAG1 | Anonymous SMB share `wave` (smbclient, guest login) | `RmxhZzF7V2VsY29tZV9UMF9USEUtVzNTVC1XMUxELUIwcmRlcn0KdXNlcjp3YXZleApwYXNzd29yZD9wZW4K` |
| FLAG2 | Home directory of `wavex` after reviewing `/usr/share/av/westsidesecret` | `Flag2{Weeeeeeeeeeeelcome to my ctfd}` |

## Attack Path Summary

1. Nmap identified SSH, HTTP, and Samba (139/445) open, with guest SMB access permitted.
2. enum4linux scoped out further SMB/NetBIOS enumeration options.
3. Anonymous `smbclient` login to the `wave` share exposed `FLAG1.txt` directly — no authentication required.
4. Decoding FLAG1 (Base64) revealed a username and password hint, enabling an SSH foothold as `wavex`.
5. A writable-path sweep uncovered a suspicious script planted in a fake antivirus directory.
6. Reviewing that location led to `FLAG2`, completing the exercise.

## Lessons & Hardening Notes

- Guest/anonymous SMB access should be disabled unless explicitly required; shares should never host sensitive files without authentication.
- SMB message signing should be **enforced**, not just enabled-but-not-required, to reduce relay attack exposure.
- Files/scripts in misleadingly named system paths (e.g. a fake `/usr/share/av` directory) are a reminder to audit non-standard directories under shared system paths.
- Disabling unused services (`smbd`/`nmbd`) when not needed reduces attack surface significantly.

---

*Performed in an isolated lab environment for educational purposes.*
