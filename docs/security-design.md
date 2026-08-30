# XYVEX Security Model & Design Specifications

This document details the security architecture, cryptographic primitives, authorization controls, and secret isolation mechanisms engineered into **XYVEX**.

---

## 1. Authentication & Token Management

XYVEX uses an enterprise-grade stateless JWT authentication strategy coupled with stateful database refresh token rotation.

```text
┌──────────────┐         POST /api/v1/auth/login          ┌──────────────┐
│  Web Client  ├─────────────────────────────────────────►│  FastAPI App │
└──────┬───────┘                                          └──────┬───────┘
       │                                                         │
       │  Returns Access Token (15 min) + HTTPOnly Refresh Cookie│
       │◄────────────────────────────────────────────────────────┘
       │
       │  Subsequent Requests: Authorization: Bearer <Access Token>
       ├─────────────────────────────────────────────────────────►
       │
       │  On Expiry (401): POST /api/v1/auth/refresh
       ├─────────────────────────────────────────────────────────►
       │  (Rotates Refresh Token in DB & returns new Access Token)
```

### Key Security Specifications

* **Access Tokens**: Short-lived JSON Web Tokens (15-minute expiration) signed with HMAC-SHA256 (`HS256`). Encodes user UUID, permissions, and token ID (`jti`).
* **Refresh Tokens**: Long-lived (7 days) cryptographically random UUID tokens stored hashed in the PostgreSQL database.
* **Token Rotation**: Every refresh request revokes the old refresh token and issues a new pair. If a revoked refresh token is re-used, the system triggers an immediate security alert and revokes *all* active sessions for that user (reuse detection).
* **Password Hashing**: Passwords are hashed using **Argon2id** (or PBKDF2-HMAC-SHA256 fallback) with random 128-bit salts, enforcing high memory/time computational cost to prevent offline GPU cracking.

---

## 2. Multi-Tenant Authorization & Isolation

A critical requirement in XYVEX is preventing horizontal privilege escalation between users.

### The Uniform 404 Isolation Strategy

To prevent attackers from discovering the existence of another user's challenges or resources via ID enumeration (`GET /api/v1/challenges/1094`):

```python
# Conceptual Authorization Enforcement Pattern
async def get_user_challenge(challenge_id: UUID, current_user: User, db: AsyncSession):
    query = select(Challenge).where(
        Challenge.id == challenge_id,
        Challenge.user_id == current_user.id # Strict owner scoping
    )
    result = await db.execute(query)
    challenge = result.scalar_one_or_none()
    
    if not challenge:
        # Uniform 404 response regardless of whether the ID exists in another account
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND, 
            detail="Challenge not found"
        )
    return challenge
```

> [!IMPORTANT]
> The API **never** returns `403 Forbidden` for resource ownership checks. A `403` leaks that a resource ID exists. Returning `404 Not Found` completely eliminates resource enumeration vectors.

---

## 3. Cryptography & Sensitive Data at Rest

In CTFs and penetration tests, users collect sensitive target passwords, private SSH keys, and proof flags. XYVEX enforces field-level symmetric encryption for all sensitive fields.

```text
Plaintext Secret Payload ("P@ssw0rd123!")
                    │
                    ▼
      Fernet Encryption Engine
   (AES-128-CBC + HMAC-SHA256 Signature)
   Key: XYVEX_DATA_ENCRYPTION_KEY
                    │
                    ▼
Base64 Ciphertext stored in PostgreSQL
("gAAAAABmX...29xK0_v3A==")
```

* **Encryption Standard**: Fernet symmetric encryption implementation from standard Python `cryptography` primitives.
* **Key Hierarchy**: Master key (`XYVEX_DATA_ENCRYPTION_KEY`) is injected via environment variables at container runtime and never persisted in database tables or code repos.
* **Explicit Reveal Flow**: API payloads return credentials in masked format (`••••••••`). Unmasking requires an explicit client request (`POST /api/v1/credentials/{id}/reveal`) that logs an audit event.

---

## 4. Email Verification & Password Reset Security

1. **Email Verification**:
   - Accounts are created in an unverified state (`is_verified = False`).
   - Generates a single-use cryptographically signed URL containing a 256-bit token with a 24-hour expiration.
   - Dispatches HTML verification emails via the Resend API.

2. **Password Reset Flow**:
   - Accepts password reset requests without leaking whether an email address exists in the system ("If an account exists, a reset link has been sent").
   - Invalidates all existing active sessions and refresh tokens upon successful password reset.

---

## 5. Automated Secret Redaction Filter

To ensure users do not accidentally publish target credentials or private keys when exporting markdown writeups, XYVEX executes a streaming multi-pass redaction filter:

| Pattern Type | Detection Mechanism | Replacement Placeholder |
| :--- | :--- | :--- |
| **Captured Passwords** | Match against database `Credential.secret_data` | `[REDACTED_PASSWORD]` |
| **SSH Private Keys** | Regex `-----BEGIN (RSA\|OPENSSH) PRIVATE KEY-----` | `[REDACTED_PRIVATE_KEY]` |
| **API Keys / Tokens** | Regex matching AWS, JWT, and generic Bearer patterns | `[REDACTED_TOKEN]` |
| **HTB/THM Flags** | Regex `[a-f0-9]{32}` / `(HTB\|THM)\{[^}]+\}` | `[REDACTED_FLAG]` |
