# Enterprise Verification (`enterprise_verify`)

> Business-license KYC: user uploads a license photo → SiliconFlow
> DeepSeek-OCR extracts fields → AI auto-audit → high-confidence auto-
> approve / otherwise manual review.

- **Identifier:** `enterprise_verify`
- **Version:** 1.5.3
- **Category:** system
- **Menu group:** Users & Support → Enterprise Verification
- **Min app version:** 0.10.0
- **Compatible editions:** enterprise, standard, pro, finance, minipro
- **Agent role:** `service`
- **Python deps:** none

---

## Core Features

### 1. OCR (`services.py`, 6.9 KB)
- Calls SiliconFlow DeepSeek-OCR API (`siliconflow_api_key`).
- Extracts from license photo: company name / unified social credit code /
  legal rep / capital / established date / business scope.
- `max_retry=3`.

### 2. AI auto-audit
- High-confidence + `auto_approve=true` → auto-approve.
- Otherwise manual review queue.

### 3. 7 HTTP endpoints
- **User**: `POST /ocr` (upload + recognize), `POST /submit`,
  `GET /status`.
- **Admin**: `GET /` (list), `POST /<id>/approve`,
  `POST /<id>/reject`, `GET/POST /settings`.

### 4. 3 hooks
- `enterprise_verify/submit`, `enterprise_verify/audit`,
  `enterprise_verify/ocr_recognize`.

---

## At a glance

| Area | Value |
|---|---|
| Storage | PG schema `enterprise_verify`, **2 tables** |
| Blueprints | `/admin/enterprise-verify` (`routes_admin.py`, 7.6 KB) + `/api/enterprise-verify` (`routes_user.py`, 6.1 KB) |
| HTTP endpoints | **7** (3 user + 4 admin) |
| Service layer | `services.py` (6.9 KB) |
| Admin UI | `templates/admin_verify.html` (4.2 KB) |
| Python deps | none |
| Permissions | `api:read`, `api:write`, `network:request`, `admin:access` |
| Capabilities | `enterprise.verify.ocr`, `enterprise.verify.audit` |
| Hooks provided | `enterprise_verify/submit`, `enterprise_verify/audit`, `enterprise_verify/ocr_recognize` |
| Dashboard | `pending`, `approved`, `rejected` |

### 2 tables

| Table | Purpose |
|---|---|
| `enterprise_verifications` | Applications (OCR fields + audit status) |
| `agent_registry` | Agent registry |

### Config (3 items)

| Key | Default | Purpose |
|---|---|---|
| `siliconflow_api_key` | empty | SiliconFlow DeepSeek-OCR API Key |
| `auto_approve` | false | Auto-approve high-confidence results |
| `max_retry` | 3 | Max OCR retries |

---

## Five-minute quick start

1. Enable — auto-creates 2 tables.
2. Open **Users & Support → Enterprise Verification**.
3. Fill `siliconflow_api_key`.
4. Optionally enable `auto_approve`.
5. Users upload license → OCR → auto / manual audit.

---

## Documents

| Document | Audience | Location |
|---|---|---|
| This file | End user / integrator | `README.md` |
| Technical whitepaper | Developers | [`docs/whitepaper.md`](docs/whitepaper.md) |
| User guide / operation manual | Ops / KYC | [`docs/user-guide.md`](docs/user-guide.md) |
| Changelog | All | [`CHANGELOG.md`](CHANGELOG.md) |
