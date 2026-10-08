# TryHackMe: Network Services (Room 1)

**Platform:** TryHackMe
**Difficulty:** Easy
**Date completed:** October 2026
**Target:** Single Linux lab machine, three services in scope: SMB, Telnet, FTP

## Overview

This room covers enumeration and exploitation of three classic, commonly
misconfigured network services. Rather than relying on a software
vulnerability, each service here is beaten through poor configuration —
anonymous access, a hidden "backdoor" service, and a weak credential. That
makes it a good room for building enumeration discipline: in each case, the
win comes from reading the output carefully, not from running an exploit.

## Environment setup

I worked this room from my local Windows machine over the TryHackMe
OpenVPN client rather than the browser AttackBox, after running out of
AttackBox free-tier time. That brought its own setup work before I could
even start on the target:

- **Tooling:** Installed `nmap` on Windows via `winget install --id=Insecure.Nmap -e`. Had to restart the terminal session afterward — an open shell doesn't pick up a PATH change made by an installer run in a different window.
- **Npcap driver issue:** Nmap initially failed with `Error compiling our pcap filter: expression rejects all packets`, caused by an outdated driver on the OpenVPN virtual adapter. Fixed properly by updating Npcap and running the terminal as Administrator. Before that fix, `-Pn --unprivileged` worked as a workaround by forcing Nmap onto standard user-mode sockets instead of raw packet capture.
- **Hydra on Windows:** Windows Defender flagged a locally-built Hydra as a Trojan/HackTool, which is expected behavior for a password-cracking tool, not a sign anything was wrong. Used a pre-compiled Windows Hydra build instead and added a Defender folder exclusion for a dedicated `C:\Tools` directory so the binary wouldn't get quarantined mid-attack.
- **Linux-path assumptions:** Commands copied from Linux-oriented guides reference paths like `/usr/share/wordlists/rockyou.txt`, which don't exist on Windows. Pulled a copy of `rockyou.txt` into the local tools directory and adjusted syntax to use relative paths (`.\hydra.exe ...`) instead.

Worth noting for anyone repeating this on Windows: a lot of the room's
friction had nothing to do with the target machine and everything to do
with running offensive Linux tooling on a Windows host. That's a fair
reflection of real-world work too — your attack environment is rarely
pre-built for you.

## Methodology

### SMB

- **Enumeration:** An initial `nmap` scan showed three open ports, with SMB running on the expected 139 and 445. Ran `enum4linux` for full basic enumeration, which returned the default `WORKGROUP` name, a machine name of `POLOSMB`, and an OS version string. One of the shares returned stood out from the rest as worth investigating further, named `profiles`.
- **Anonymous access:** Connected to the `profiles` share with `smbclient`, authenticating as `Anonymous` with no password. The share allowed the connection without credentials — a straightforward anonymous-access misconfiguration.
- **What was inside:** The share contained a profile folder belonging to a specific user. Inside, an `.ssh` directory held the user's SSH keys, including a private key (`id_rsa`) that could be used to authenticate as that user.
- **Access:** Downloaded the private key locally, set its permissions to `600` (SSH refuses to use a key with overly-open permissions), and used it together with the username recovered from the share to authenticate over SSH.

### Telnet

- **Enumeration:** A full-range `nmap` scan (`-p-`) found exactly one open port, on an unusual high port rather than the standard Telnet port 23. Re-running the scan without `-p-` (i.e. the default top-1000-ports scan) showed zero open ports — the service was only visible because it sat outside Nmap's default port range entirely. That's the main lesson of this section: a default scan would have missed this host's only open service completely.
- **Banner/service identification:** Nmap still identified the protocol on that port as TCP, and connecting to it directly returned a banner naming itself as a "backdoor" and referencing a specific username, which gave a likely account to target.
- **Initial connection:** Connecting via `telnet` opened a session, but typing commands produced no visible output — no obvious confirmation the service was actually running them.
- **Confirming command execution:** Set up a local `tcpdump` listener for ICMP traffic, then sent a `ping` command through the Telnet session targeting my own machine. Receiving the ping confirmed the session really was executing system commands — it just wasn't echoing output back to the terminal.
- **Getting a shell:** Generated a reverse shell payload with `msfvenom` (a Netcat-based Unix payload) targeting my OpenVPN IP and a chosen listening port, started a Netcat listener locally, then pasted the payload into the Telnet session to execute it. That returned a working shell on the target.

### FTP

- **Enumeration:** Ran `nmap` against the target (`10.128.179.138` from my session) to confirm the FTP port and service banner, which revealed the FTP server variant in use.
- **Anonymous access:** Logged in with username `anonymous` and no password, per FTP's common (and insecure) anonymous-login convention. Found a file in the anonymous directory that pointed toward a likely username, `mike`.
- **Credential attack:** Used Hydra to brute-force the password for `mike` over FTP, pointed at the `rockyou.txt` wordlist:
  ```
  .\hydra.exe -t 4 -l mike -P rockyou.txt -vV 10.128.179.138 ftp
  ```
- **Access:** Logged in via FTP as `mike` with the cracked credential and retrieved the target file to confirm the compromise.

## Key findings

- **SMB:** Anonymous/unauthenticated access to a share exposed information that shouldn't be available without credentials — including, ultimately, a path to a private key.
- **Telnet:** A service was deliberately hidden on a non-standard port rather than secured, which only delays discovery rather than preventing it, and the service itself ran as an unauthenticated command interface.
- **FTP:** Anonymous login was enabled and a weak, dictionary-crackable password was in use on a real account, letting a brute-force attack succeed in reasonable time.

Across all three, the pattern is the same: the services weren't exploited
through a technical flaw in the software, but through configuration that
trusted the network more than it should have.

## Defensive takeaways

- **SMB:** Disable anonymous and guest access, enforce SMB signing, disable SMBv1, and restrict shares to least privilege. Enumeration tools like `enum4linux` only work because unauthenticated users are allowed to ask these questions in the first place.
- **Telnet:** Telnet transmits everything, including credentials, in cleartext, and "hiding" a service on a non-standard port is not a substitute for securing it. Replace it with SSH; if it must remain for legacy reasons, restrict it to a management VLAN and alert on connection attempts.
- **FTP:** Disable anonymous login unless there's a genuine business need for it. Use SFTP or FTPS instead of plain FTP, enforce strong passwords with lockout/rate-limiting to blunt brute-force attempts, and don't expose FTP directly to the internet.
- **Across all three:** Run only the services actually needed, patch and update regularly, segment the network so a compromised service doesn't expose everything else, and log authentication attempts so repeated failures (like a Hydra run) get flagged rather than going unnoticed.

## What I learned

The biggest lesson here came from getting my own attack environment working on Windows instead of a pre-built Linux AttackBox. Tool installation, driver issues, and even antivirus false-positives on security tools are things a lot of writeups
skip over, but they're a realistic part of doing this work outside a sandboxed lab. On the target side, the Telnet section was the most instructive: running a full port-range scan instead of relying on defaults was the only reason that service was found at all, which reinforced that thorough enumeration matters more than clever exploitation.
