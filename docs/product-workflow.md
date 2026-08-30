# XYVEX Product Workflow & Reusable Knowledge Engine

This document details the core product workflows, domain lifecycle models, and the proprietary **Reusable Knowledge Engine** implemented in **XYVEX**.

---

## 1. The Authorized CTF Workflow Lifecycle

XYVEX is built specifically for security researchers, penetration testers, and CTF players. The system structures what is traditionally a chaotic mix of terminal buffer histories, temporary text files, and unorganized screenshots into an indexed domain hierarchy.

```text
┌────────────────────────────────────────────────────────────────────────┐
│                        1. Target Initialization                        │
│   Create Challenge → Specify Target IP (10.10.11.35) & Platform        │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│                      2. Host & Service Enumeration                     │
│   Ingest Port Scans (Nmap/Rustscan) → Auto-populate Hosts & Ports      │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│                   3. Interactive Command Workspace                     │
│   Execute Commands → Track Attempts → Tag Outcomes (Success / Fail)   │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│                  4. Evidence & Credential Capture                      │
│   Store Loot → Encrypt Passwords/Proofs → Attach terminal snippets    │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│               5. Knowledge Extraction & Writeup Export                 │
│   Promote Commands to Library → Generate Redacted Markdown Writeup    │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Domain Data Model Hierarchy

Every security investigation within XYVEX follows a deterministic parent-child hierarchy:

1. **Challenge**: The top-level container (e.g. `HTB: Cicada`). Holds target IP, platform metadata, global notes, user checklists, and final writeups.
2. **Host**: An IP or hostname under evaluation (e.g. `10.10.11.35` / `cicada.htb`).
3. **Service**: An open port/protocol discovered on a host (e.g. `Port 445 / TCP / SMB`).
4. **Command**: A specific CLI string run against a service (e.g. `smbclient -L //10.10.11.35 -N`).
5. **Execution Attempt**: A recorded execution instance containing exit code, stdout/stderr logs, timestamp, and operator tags (`Success`, `Exploit-Attempt`, `PrivEsc`).
6. **Evidence / Proof**: Screen captures, flag hashes, or extracted database dumps tied directly to an execution attempt.
7. **Writeup**: A markdown report dynamically assembled from selected successful attempts and evidence logs.

---

## 3. The Reusable Knowledge Engine

A central innovation in XYVEX is solving the **"volatile knowledge problem"** in cybersecurity labs: security practitioners frequently re-invent command syntaxes across different challenges because previous command logs are scattered.

### How Parameterization Works

When an operator marks a command attempt as `Successful` or clicks **"Promote to Library"**, XYVEX executes an automated pattern substitution pipeline:

#### Step 1: Raw Command Input
```bash
smbclient -L //10.10.11.35 -U "john%Password123!" -W CORP
```

#### Step 2: AST / Pattern Parser Tokenization
The parser evaluates hardcoded arguments against the active challenge domain state:
- Matches `10.10.11.35` against `Challenge.target_ip` -> Replaces with `<TARGET_IP>`
- Matches `john` against stored target usernames -> Replaces with `<USERNAME>`
- Matches `Password123!` against stored passwords -> Replaces with `<PASSWORD>`
- Matches `CORP` against active domain names -> Replaces with `<DOMAIN>`

#### Step 3: Parameterized Library Template Creation
```bash
smbclient -L //<TARGET_IP> -U "<USERNAME>%<PASSWORD>" -W <DOMAIN>
```

#### Step 4: Metadata Tagging & Indexing
The snippet is automatically indexed with search tags derived from the service context:
- **Category**: `Enumeration / Active Directory`
- **Protocol**: `SMB / CIFS`
- **Port**: `445`
- **OS**: `Windows`

---

## 4. Contextual Recommendation System

When a user creates a new challenge or adds a new service (e.g. Port 445 on Target `10.10.11.90`):

1. The frontend queries `/api/v1/library/recommendations?service=smb&port=445&os=windows`.
2. XYVEX ranks library templates based on past success rates and usage frequency.
3. The UI presents recommended commands in a 1-click execution drawer:

```text
[RECOMMENDED] Technique for Port 445 (SMB):
   [smbclient Null Session Enumeration]
   Command: smbclient -L //<TARGET_IP> -N

   [Copy with Active IP (10.10.11.90)]
```

When clicked, `<TARGET_IP>` is dynamically substituted with the new challenge target (`10.10.11.90`), allowing instantaneous execution without manual typing.

---

## 5. Automated Writeup Synthesis & Redaction

At challenge completion, XYVEX provides a single-click Writeup Synthesis engine:

1. **Auto-Assembles Timeline**: Pulls all `Successful` command attempts in chronological order.
2. **Structures Sections**: Automatically formats markdown headers for Reconnaissance, Initial Access, Privilege Escalation, and Proof of Concept.
3. **Secret Redaction**: Before rendering or exporting the markdown writeup, the backend runs the **Redaction Engine**, replacing active credentials, private keys, and session tokens with sanitized place-holders (`[REDACTED_PASSWORD]`, `[REDACTED_FLAG]`).
