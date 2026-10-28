# Social Push (`social_push`)

> Multi-platform social content publishing. One draft → push to WeChat OA,
> Weibo, Toutiao, Twitter/X, LinkedIn, Reddit, Telegram. Publish history +
> OAuth callbacks.

- **Identifier:** `social_push`
- **Version:** 2.0.0
- **Category:** social
- **Menu group:** Publishing → Social Push
- **Min app version:** 0.10.0
- **Compatible editions:** enterprise, standard, pro, finance, minipro
- **Agent role:** `content`
- **Python deps:** `praw` (Reddit), `tweepy` (Twitter)

---

## Core Features

### 1. 7 channel adapters
- **CN**: WeChat OA, Weibo, Toutiao.
- **INTL**: Twitter/X (tweepy), LinkedIn, Reddit (praw), Telegram Bot.

### 2. Publish queue (`services/queue.py`, 8.1 KB)
- Async publishing.

### 3. Publish history (`models.py`, 19.3 KB)
- `social.push.history` capability.

### 4. OAuth callbacks (`routes_oauth.py`, 11.7 KB)
- Per-platform OAuth callback.

### 5. Hooks
- Provided: `social_push/publish`.

---

## At a glance

| Area | Value |
|---|---|
| Blueprints | `/admin/social-push` (`routes.py`, 45.8 KB) + `/oauth` (`routes_oauth.py`, 11.7 KB) |
| Admin UI | `templates/admin_socialpush.html` (51.3 KB) |
| Queue | `services/queue.py` (8.1 KB) |
| Python deps | praw, tweepy |
| Permissions | social_push.read/write/publish, network:request |
| Capabilities | social.push.publish, social.push.history |
| Hook provided | social_push/publish |
| Dashboard | total_publishes / today_publishes |

### Config (20 items across 7 channels, secrets writeOnly)

| Channel | Keys |
|---|---|
| WeChat OA | wechat_app_id / wechat_app_secret |
| Weibo | weibo_app_key / weibo_access_token |
| Toutiao | toutiao_app_id / toutiao_access_token |
| Twitter | twitter_api_key / twitter_api_secret / twitter_access_token / twitter_access_secret / twitter_bearer_token |
| LinkedIn | linkedin_client_id / linkedin_client_secret / linkedin_access_token |
| Reddit | reddit_client_id / reddit_client_secret / reddit_username / reddit_password |
| Telegram | telegram_bot_token / telegram_channel |

---

## Five-minute quick start

1. Enable — install praw / tweepy.
2. Open **Publishing → Social Push**.
3. Fill AppID / Secret per platform or run OAuth callback.
4. Compose draft → pick channels → publish.
5. View publish history.

---

## Documents

| Document | Audience | Location |
|---|---|---|
| This file | Marketing / content ops | `README.md` |
| Technical whitepaper | Developers | [`docs/whitepaper.md`](docs/whitepaper.md) |
| User guide / operation manual | Ops | [`docs/user-guide.md`](docs/user-guide.md) |
| Changelog | All | [`CHANGELOG.md`](CHANGELOG.md) |
