# Vault (`vault`)

> Data vault. Full / incremental backups, AES-256-GCM encryption, gzip
> compression, scheduled backups, multi-target storage (local / S3 / OSS /
> Azure / GCS / SFTP), audit log.

- **Identifier:** `vault`
- **Version:** 2.6.5
- **Category:** system
- **Menu group:** System → Vault
- **Min app version:** 0.10.0
- **Compatible editions:** enterprise, standard, pro, finance, research, minipro
- **Agent role:** `ops`
- **Python deps:** croniter, cryptography, paramiko, requests
- **Optional:** boto3 (S3), oss2 (Alibaba OSS), azure-storage-blob, google-cloud-storage

---

## Core Features

### 1. Backup lifecycle
- `POST /api/create`, `GET /api/list`, `GET /api/download/<label>`,
  `DELETE /api/delete/<label>`, `DELETE /api/cleanup`.
- `vault.create_backup` / `vault.list_backups` capabilities.

### 2. Restore
- `GET /restore` page.
- `vault.restore` capability.

### 3. Schedules
- `GET/POST /api/schedule/list`, `POST /api/schedule/create`,
  `PUT/DELETE /api/schedule/<id>`, `POST /api/schedule/<id>/toggle`.
- `interval_hours=24`, `schedule.enabled=false` by default.

### 4. Multi-target storage
- `GET/POST /api/storage/list`, `PUT/DELETE /api/storage/<id>`,
  `POST /api/storage/<id>/test`.
- Local / S3 / OSS / Azure Blob / GCS / SFTP (paramiko).

### 5. Encryption / compression
- AES-256-GCM, `key_source=env`.
- gzip level=6.

### 6. Audit
- `GET /audit`, `GET /api/audit`.
- `vault.audit` capability.

### 7. Health check
- `GET /api/health`.
- `vault/health_check` hook.

---

## At a glance

| Area | Value |
|---|---|
| Blueprint | `/admin/vault` (`routes.py`) |
| HTTP endpoints | **30** |
| Python deps | croniter, cryptography, paramiko, requests |
| Optional | boto3, oss2, azure-storage-blob, google-cloud-storage |
| Permissions | vault.read/write/download/admin |
| Capabilities | vault.create_backup, vault.list_backups, vault.restore, vault.audit |
| Hooks provided | vault/create_backup, vault/list_backups, vault/delete_backup, vault/health_check, vault/audit_log |
| Dashboard | total_backups / success_backups / total_schedules |

### Key config

| Key | Default | Purpose |
|---|---|---|
| `backup_dir` | data/vault | Local backup dir |
| `keep_days` | 30 | Retention days |
| `include_files` | true | Include files |
| `include_config` | true | Include config |
| `encryption.enabled` | true | Encryption on/off |
| `encryption.algorithm` | aes256-gcm | Algorithm |
| `encryption.key_source` | env | Key source (env var) |
| `compression.algorithm` | gzip | Compression |
| `compression.level` | 6 | Level |
| `storage.type` | local | Default storage type |
| `schedule.enabled` | false | Schedule on/off |
| `schedule.interval_hours` | 24 | Interval hours |

---

## Five-minute quick start

1. Enable — install croniter / cryptography / paramiko.
2. Open **System → Vault**.
3. Set encryption key (env var).
4. Add storage target (local / S3 / OSS) → test.
5. Create schedule → run once.

---

## Documents

| Document | Audience | Location |
|---|---|---|
| This file | Ops / SRE | `README.md` |
| Technical whitepaper | Developers | [`docs/whitepaper.md`](docs/whitepaper.md) |
| User guide / operation manual | Ops | [`docs/user-guide.md`](docs/user-guide.md) |
| Changelog | All | [`CHANGELOG.md`](CHANGELOG.md) |
