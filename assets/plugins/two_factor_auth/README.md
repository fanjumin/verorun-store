# Two-Factor Authentication (`two_factor_auth`)

> TOTP-based 2FA for VeroRun. Compatible with Google / Microsoft
> Authenticator. Recovery codes + audit + failure lockout.

- **Identifier:** `two_factor_auth`
- **Version:** 1.2.0
- **Category:** system
- **Menu group:** Security & Compliance → 2FA Settings
- **Min app version:** 0.10.0
- **Compatible editions:** enterprise, standard, pro, finance, research, minipro
- **Agent role:** `service`
- **Python deps:** pyotp, qrcode, bcrypt, cryptography

---

## Core Features

### 1. TOTP setup
- `POST /setup-token`, `GET /setup-page`, `POST /setup/init`,
  `POST /setup/verify`, `POST /setup/disable`.
- `two_factor.setup` capability.

### 2. Login challenge
- `GET /challenge-page`, `POST /challenge/verify`.
- `two_factor.verify` capability.

### 3. Recovery codes
- `recovery_code_count=10`.
- `two_factor.recovery` capability.

### 4. Status
- `GET /status`, `GET /health`.

### 5. Anti-bruteforce
- `max_failed_attempts=5`.
- `lockout_seconds=900` (15 min).

### 6. TOTP tolerance
- `totp_valid_window=1` (±30s tick).
- `issuer_name=VeroRun`.

---

## At a glance

| Area | Value |
|---|---|
| Storage | PG schema `two_factor_auth` |
| Blueprint | `/plugin/two_factor_auth` (`routes.py`) |
| HTTP endpoints | **9** |
| Python deps | pyotp, qrcode, bcrypt, cryptography |
| Permissions | user:profile, routes (admin:access) |
| Capabilities | two_factor.setup, two_factor.verify, two_factor.recovery |

### Config (5 items)

| Key | Default | Purpose |
|---|---|---|
| `max_failed_attempts` | 5 | Lockout threshold |
| `lockout_seconds` | 900 | Lockout seconds |
| `recovery_code_count` | 10 | Number of recovery codes |
| `issuer_name` | VeroRun | Authenticator display name |
| `totp_valid_window` | 1 | TOTP tolerance window |

---

## Five-minute quick start

1. Enable — install pyotp / qrcode / bcrypt / cryptography.
2. Open **Security & Compliance → 2FA Settings**.
3. Scan QR with Authenticator.
4. Verify 6-digit code.
5. Save recovery codes.

---

## Documents

| Document | Audience | Location |
|---|---|---|
| This file | End users / security admins | `README.md` |
| Technical whitepaper | Developers | [`docs/whitepaper.md`](docs/whitepaper.md) |
| User guide / operation manual | Users | [`docs/user-guide.md`](docs/user-guide.md) |
| Changelog | All | [`CHANGELOG.md`](CHANGELOG.md) |
