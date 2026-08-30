# XYVEX Application Screenshots Showcase

This directory contains sanitized screenshot captures demonstrating the user interface, layout density, and workflows of **XYVEX**.

---

## 🔒 Screenshot Sanitization Rules

To ensure strict operational security and privacy when showcasing product screenshots publicly, all screenshots in this directory must strictly comply with the following sanitization criteria:

### 1. Zero Sensitive Data Exposure
- **Forbidden**: Real user email addresses, private production API keys (e.g. Resend keys), production JWT tokens, database passwords, or operational secrets.
- **Allowed**: Standard synthetic demo user profiles (`operator@xyvex.internal` or `demo@xyvex.lab`).

### 2. Standardized Lab Targets
- All screenshots use public, synthetic, or standard training lab targets:
  - **Challenge Target**: `Cicada` (Hack The Box)
  - **Target IP**: `10.10.11.35`
  - **Domain**: `cicada.htb` / `CORP.LOCAL`

### 3. Redacted Evidence & Flag Proofs
- Real CTF root/user flag hashes and sensitive target credentials must be masked or replaced with synthetic demonstration hashes (`htb{s4n1t1z3d_c4s3_stUdy_fL4g!}`).

---

## 📷 Screenshot Manifest & Descriptions

| Filename | View / Component | Key Interface Features Demonstrated |
| :--- | :--- | :--- |
| `overview.png` | **Dashboard Overview** | Active target metrics, platform distribution charts, recent command attempt timeline, and quick-action challenge launcher. |
| `boxes.png` | **Target Hub (Boxes)** | Host reconnaissance manager, open service port listing (Nmap parse ingestion), OS tag badges, and target credential inventory. |
| `workspace.png` | **Terminal Workspace** | Real-time command execution log stream, outcome tagging (Success/Failure), execution timestamping, and output snippet viewer. |
| `library.png` | **Knowledge Library** | Reusable command snippet repository, protocol category filters (SMB, HTTP, SSH, Kerberos), search bar, and parameter template preview. |
| `writeup.png` | **Writeup Generator** | Split-pane Markdown editor, dynamic timeline evidence linking, and automated secret redaction filter toggle. |
| `settings.png` | **Security & Account** | Active session token manager, database encryption status indicator, email verification state, and dark mode theme preferences. |
