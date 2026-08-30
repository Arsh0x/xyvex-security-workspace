# XYVEX Application Screenshots Showcase

This directory contains sanitized screenshot captures demonstrating the user interface, layout density, and workflows of **XYVEX**.

---

## Screenshot Sanitization Rules

To ensure strict operational security and privacy when showcasing product screenshots publicly, all screenshots in this directory strictly comply with the following sanitization criteria:

### 1. Zero Sensitive Data Exposure
- **Forbidden**: Real user email addresses, private production API keys (e.g. Resend keys), production JWT tokens, database passwords, or operational secrets.
- **Allowed**: Standard synthetic demo user profiles (`operator@xyvex.internal` or `demo@xyvex.lab`).

### 2. Standardized Lab Targets
- All screenshots use public, synthetic, or standard training lab targets:
  - **Challenge Targets**: `Cicada` (Hack The Box, IP: `10.10.11.35`), `BoardLight` (TryHackMe)
  - **Domain**: `cicada.htb` / `CORP.LOCAL`

### 3. Redacted Evidence & Flag Proofs
- Real CTF root/user flag hashes and sensitive target credentials are replaced with synthetic demonstration hashes (`htb{s4n1t1z3d_c4s3_stUdy_fL4g!}`).

---

## Screenshot Manifest & Descriptions

| Filename | View / Component | Key Interface Features Demonstrated |
| :--- | :--- | :--- |
| `overview.png` | **Workspace Dashboard** | Active target metrics (`BoardLight`, `Cicada`), box pipeline count, Knowledge Library stats (128 techniques / 342 commands), recent useful commands drawer, and workspace activity feed. |
| `boxes.png` | **Worklog & Service Hub** | Box phase tracking (Service Enumeration), interactive attack workflow chart, discovered services table (`SSH:22`, `HTTP:80`, `SMB:445`, `WinRM:5985`), recent worklog timeline, quick capture note drawer, and host context. |
| `recon.png` | **Reconnaissance & Scan Ingestion** | Target information sidebar (`10.10.11.35`, `cicada.htb`), interactive Nmap CLI command buffer, syntax-highlighted scan output log, open ports list, discovered credentials, and saved scan artifacts (`nmap.txt`, `gobuster.txt`). |
| `checklist.png` | **Methodology & Progress Engine** | Multi-phase security methodology checklist (Reconnaissance 4/5, Exploitation 2/4, Privilege Escalation 1/4), progress donut chart (35% Complete), rabbit hole warning alerts, missing step suggestions, and helpful tips. |
| `library.png` | **Technique Library & Parameterization** | Reusable offensive technique repository (SMB Share Enumeration, Kerberos User Enumeration, DNS Zone Transfer, AS-REP Roasting), category filters, difficulty tags, OS compatibility badges, reusable command drawer, common mistakes, and target box references. |
