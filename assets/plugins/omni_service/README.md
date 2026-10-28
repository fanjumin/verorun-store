# Omni Service

Omnichannel customer-service inbox for VeroRun. Inbound messages from every
supported channel are normalized into **one local conversation model**, answered
first by the AI (through the existing chatbot engine), and handed off to a human
agent from a single console window that always shows **which channel a message
came from**.

This is a commercial add-on. Version composition is driven by plugin bundles
(`plugin_manager/pricing.py` → `bundles`), not by edition YAML.

## What it does

| Capability | Detail |
|---|---|
| `omni.inbound.receive` | One webhook per channel, signature-verified, normalized to `OmniEvent` |
| `omni.conversation.list` | Cross-channel inbox with status / channel / assignee filters |
| `omni.conversation.assign` | Take over, release back to AI, close |
| `omni.message.list` | Per-conversation message log with media and channel provenance |
| `omni.message.send` | Agent reply, with platform push-window enforcement |
| `omni.handoff.create` | Deterministic escalation ledger (keyword / fail threshold / manual) |
| `omni.channel.status` | Which channels are wired and configured |

## 2026-10-21 updates

- **Desktop edition converged into `omni-service`.** `plugin.json` declares
  `compatible_editions: ["enterprise", "omni-service"]`. Per the 2026-10-21 decision,
  the standalone `omni-desktop` edition is deprecated in favour of a server + thin
  shell model: the plugin set runs on the cloud server
  (`deploy/editions/omni-service.yaml`) and the desktop shell embeds no plugin payload,
  wiring remotely to the user's public admin API.
- **Handoff intake form (`handoff_form.py`).** With `handoff_form_enabled` and
  `handoff_form_schema` set, the bot asks plain-text questions through the same channel
  before a human picks up. Answers land in `omni_service.handoff_form_submissions`
  (`migrations/v1.1.0__handoff_form.sql`); a completed form is summarised into the
  thread and files a ticket through `create_ticket_from_chat()` only when the handoff
  has no ticket yet.

## Channel support

| Channel | Mode | Inbound | Outbound | Notes |
|---|---|---|---|---|
| `wecom_group` 企业微信群 @机器人 | push | ✅ | ✅ | AES-verified callback; group replies go through the group-robot webhook so the answer lands in the thread |
| `wechat_kf` 微信客服 | **pull** | ✅ | ✅ | Callback only carries a `Token`; bodies are fetched with `sync_msg` + persisted cursor. This is the channel for external customers 1:1. |
| `feishu` 飞书机器人 | push | ✅ | ✅ | Event subscription v2 (`im.message.receive_v1`); SHA256 signature + AES envelope + verification token; POST `challenge` handshake; no push window |
| `dingtalk` 钉钉企业内部机器人 | push | ✅ | ✅ | HTTP callback mode; header `timestamp` + `sign` (HMAC-SHA256, 1h window) + `robotCode`/`chatbotCorpId` binding; group reply via `groupMessages/send`, 1:1 via `oToMessages/batchSend`; no push window |
| `qq` QQ 机器人（Webhook 模式） | push | ✅ | ✅ | `op=13` URL verification via Ed25519 over `event_ts+plain_token`; events signed over `X-Signature-Timestamp + body`; **passive reply is 5 min / 5 per message** (`msg_id` + `msg_seq`), beyond that only an opt-in active message; `GROUP_AT_MESSAGE_CREATE` / `C2C_MESSAGE_CREATE` |
| `webhook_seen` dedup | — | ✅ | — | Platform retries never re-trigger the AI |

Later milestones (declared in `normalize.CHANNELS`, adapters not yet shipped):
`wechat_mini`, `douyin_mini`, `telegram`, `line`, `slack`, `discord`,
`whatsapp`, `web`.

> `mini_app_builder` is a **mini-program generator**, not an IM channel. The
> mini-program UI wiring is a separate workstream.

## Configuration

Channel credentials are read from `im_gateway.channel_configs` (channel
`wecom`), so one credential row serves both the outbound gateway and this
inbox. Required fields for `wecom_group`:

| Field | Purpose |
|---|---|
| `corp_id` | Tenant id; also validated against the decrypted `receive_id` |
| `agent_id` | Self-built app agent id |
| `secret` | Used to mint `access_token` for `message/send` and media fetch |
| `token` | Callback token for signature verification |
| `encoding_aes_key` | 43-char AES key for callback decryption |
| `webhook_url` | Optional group-robot webhook; **required for group replies** |
| `bot_name` | Optional; strips `@机器人名` from the message body |

`wechat_kf` uses the same row (`corp_id`, `token`, `encoding_aes_key`) plus:

| Field | Purpose |
|---|---|
| `kf_secret` | Optional dedicated 微信客服 secret; falls back to `secret` |

The self-built app must also be listed under 管理后台 →「微信客服 - 可调用接口的
应用」, and the customer-service accounts must be set to "通过 API 管理", otherwise
`sync_msg`/`send_msg` return a permission error. That is an admin-console step,
not something code can work around.

`feishu` reads its own row from the same table (channel `feishu`), which the
outbound gateway already uses:

| Field | Purpose |
|---|---|
| `app_id` / `app_secret` | Mint `tenant_access_token` (cached 7200s, refreshed 300s early) |
| `verification_token` | Second gate: `payload.token` / `header.token` must match |
| `encrypt_key` | Enables signature verification **and** AES decryption. Leave empty only if the app has encryption off — then there is no signature to check |
| `bot_open_id` | Optional: the bot's own `open_id`, used to drop its `@` from the text |
| `bot_name` | Optional: same, by display name (e.g. `AI客服`) |
| `domain` | Optional: `lark` switches to the international host `open.larksuite.com` |

Feishu console prerequisites (again: admin work, not code): enable 机器人,
subscribe `im.message.receive_v1`, grant `im:message` + `im:message:send_as_bot`,
and add `im:message.group_at_msg` for group @-mentions. With only the
all-message scope, un-mentioned group chatter arrives too — it is stored but not
answered.

`dingtalk` also reads its own row (channel `dingtalk`, the same one the outbound
gateway uses):

| Field | Purpose |
|---|---|
| `app_key` / `app_secret` | Client ID / Client Secret; mint `accessToken` (cached 7200s, refreshed 300s early) and verify the `sign` header |
| `robot_code` | Robot code; also used as the conversation account key. Optional — falls back to `app_key`, which is what it usually is |
| `corp_id` | Optional but recommended: rejects a callback whose `chatbotCorpId` is not this tenant. Together with `robot_code` it is the **only** body-integrity check DingTalk offers (see limitation 13) |
| `base_url` | Optional: override `https://api.dingtalk.com` |

DingTalk console prerequisites: 创建企业内部应用 → 应用能力 → 机器人 → 消息接收模式
选 **HTTP 模式**，把 `https://<你的域名>/api/v1/omni/webhook/dingtalk` 填为消息接收
地址，然后发布应用版本（`senderStaffId` / `isAdmin` 等字段**发布上线后才返回**）。
机器人回复走 `v1.0/robot/*`，需要「企业内机器人发送消息权限」。

**Do not** point the callback at the gateway's OAPI path: `topapi/message/
corpconversation/asyncsend_v2` is 工作通知, not a robot conversation reply — it
needs `agent_id`, cannot reach external members, and does not land in the thread
the customer asked in.

`qq` reads its own row (channel `qq`, the same one the outbound gateway uses):

| Field | Purpose |
|---|---|
| `app_id` | Bot AppID. Minted into `Authorization: QQBot <token>` + `X-Union-Appid`, and compared against the callback's `X-Bot-Appid` when both exist |
| `client_secret` | Bot secret for `getAppAccessToken`. Also accepted as `app_secret` (the gateway's older field name) |
| `bot_secret` | Ed25519 key material for signature verification **and** the `op=13` answer. Different secret from `client_secret`; without it every callback is refused |
| `answer_group_all` | Optional: answer `GROUP_MESSAGE_CREATE` (全量模式) too. Off by default — that event carries no "was I mentioned" signal |
| `allow_active_message` | Optional: allow active messages once the 5-minute passive window is gone. Off by default because active messages are rate-limited (60/30 qpm, 1000/day per group) and pass through platform audit |
| `api_base` | Optional: `https://sandbox.api.sgroup.qq.com` for the sandbox environment |
| `token_url` | Optional: override `https://bots.qq.com/app/getAppAccessToken` |

QQ console prerequisites: 开放平台 → 机器人 → 开发设置 填 **Webhook 回调地址**
`https://<你的域名>/api/v1/omni/webhook/qq`（端口只允许 80/443/8080/8443），订阅
`GROUP_AND_C2C_EVENT`（`1<<25`）。保存时平台会立刻发一个 `op=13` 验证请求——
插件必须已配置 `bot_secret`，否则回包签名算不出来，地址存不下来。

## Known limitations (explicit, not hidden)

1. **External customers in a customer group cannot be answered through
   `message/send`.** That API accepts enterprise member UserIDs only. Group
   replies therefore use the group-robot webhook (visible to the whole group),
   and true 1:1 external service goes through `wechat_kf` instead.
2. **Push windows are enforced, not papered over.** The window length belongs to
   the channel (`OmniChannel.window_seconds`; `wecom_group` has none, WeChat KF
   48h, WhatsApp 24h). It is computed once in `orchestrator.window_expiry()` and
   refreshed on every inbound message, so a customer who writes again restarts
   the clock; a channel without a window never wipes a stored one (`COALESCE`).
   Enforcement happens in exactly one place — `send_reply()` — which moves the
   conversation to `blocked_window` and writes a `system` ledger row the console
   can show. A broken window calculation degrades to "no window", never to a
   dropped reply.
3. **`@` detection on WeCom groups is callback-driven.** WeCom only pushes a
   group callback when the app is actually mentioned, so in practice every
   delivered group message counts as addressed. If an operator configures the
   app to receive *all* group messages, bodies without an `@` prefix are stored
   for context but **not answered** (`mention_required` policy) — that is the
   intended behaviour, not a bug.
4. **Image understanding is not wired.** Images are stored and shown to agents;
   the AI receives a `[image message]` placeholder. No fake vision.
5. **No personal WeChat.** There is no official API; hook-based approaches
   violate the platform licence. WeCom / WeChat KF only.
6. **`sync_msg` only sees the last 3 days.** If the desktop app is offline for
   longer than that, messages from the gap are gone — the platform does not
   backfill. The cursor makes this worse to recover from (a stale cursor resumes
   where it left off and simply finds nothing), so an operator who leaves the
   machine off for days must not assume the history is complete.
7. **`send_msg` is 48h / 5 messages, and HTTP 200 is not delivery.** A 200 means
   the platform accepted the message; real failures arrive later as a
   `msg_send_fail` event (not consumed yet — registered as a follow-up). Error
   `45015` is surfaced as `degraded: true` because it means "outside the
   allowance", not "bug".
8. **A human replying from the WeCom client wins.** Those replies come back
   through `sync_msg` with `origin=5`; they are recorded as `out`/`human` and
   flip the conversation to `human`, so the AI stops answering on that thread.
9. **Feishu does not tell us who is a colleague.** A human answering in the same
   thread arrives as `sender_type=user`, exactly like a customer — there is no
   `origin` field. Group threads are still safe in practice (a human answer
   rarely `@`s the bot, and only mentioned messages trigger), but a colleague who
   *does* `@` the bot will get an AI answer on top of their own. 1:1 threads are
   unaffected.
10. **Feishu group messages arrive only when mentioned** (default scope). We
   derive `mentioned` from the `mentions` array rather than from arrival, so a
   tenant that grants 获取群组中所有消息 gets context without the bot talking over
   people.
11. **The bot's own `@` needs configuration to be removed cleanly.**
   `mentions[].id.open_id` is the mentioned party, not us; without `bot_open_id`
   or `bot_name` the mention is rendered as `@名字` and only a *leading* one is
   stripped. Set one of those two fields for exact query text.
12. **Feishu attachments need `message_id` + `file_key` together** (error 234003
   otherwise), which is why the media ref stores both. Downloads are capped at
   100 MB by the platform.
13. **DingTalk's signature does not cover the body.** Unlike Feishu, `sign` is
   `base64(HMAC-SHA256(key=appSecret, msg=timestamp + "\n" + appSecret))` — it
   proves the request was minted with our appSecret, not that the payload was
   left alone. Compensation is explicit, not pretended: a 1h timestamp window
   (platform-documented) for replay, `robotCode` + `chatbotCorpId` binding to our
   own robot/tenant, and HTTPS-only exposure. `test_body_is_not_covered_by_the_
   signature` is the executable record of this weakness.
14. **DingTalk's encrypted-callback envelope is not supported.** The robot HTTP
   callback is plaintext JSON; `{"encrypt": ...}` belongs to the event-subscription
   crypto path and is rejected with 403 rather than "guessed" — a wrong AES
   layout that silently accepts garbage is worse than a clear refusal.
15. **A DingTalk group message arrives only when the bot is @-ed**, and
   `isInAtList` / `atUsers` decides whether the AI wakes up. Outbound group
   replies need `robotCode` + `openConversationId` (`conversationId`) and a bot
   that is still in the group.
16. **External members cannot be answered 1:1.** In an external group
   `senderStaffId` is empty; only the encrypted `$:LWCP_v1:$...` senderId remains,
   and `batchSend` needs a userId. Such a thread is recorded and the reply is
   refused with `degraded: true` — a human must reach the customer another way.
17. **Rich-text pictures are placeholders.** Each `picture` node needs its own
   `downloadCode` → URL round trip; the digest keeps `[图片]` and stores no media
   ref, rather than pretending one preview covers the message.
18. **`sessionWebhook` is deliberately unused.** It expires after ~1.5h, so a
   console reply hours later would fail. Everything goes through
   `groupMessages/send` / `oToMessages/batchSend` instead.
19. **Voice notes rely on the platform's own ASR.** DingTalk delivers
   `content.recognition`; when the tenant turns that off, the message has media
   and empty text, and the local Vosk pass (after download) is the only fallback.
20. **QQ's passive reply window is 5 minutes and 5 replies per message.** This is
   the harshest window of any channel here (WeChat KF 48h, WhatsApp 24h) and it
   changes the product shape: AI first line works, a human picking the ticket up
   twenty minutes later usually does not. It is modelled per *message*
   (`conversations.last_inbound_msg_id` + `reply_seq`, claimed atomically in
   `orchestrator.next_reply_seq`), not per conversation, because `msg_id` — not
   time — is what the platform asks for. Past the window the reply is refused
   with an explicit reason unless `allow_active_message` is set.
21. **QQ `window_blocks_reply = False`.** An expired window does not move the
   conversation to `blocked_window` — the conversation is still alive, only the
   passive path is gone. `send_reply()` honours this per channel, so QQ cannot be
   silently frozen the way a 48h-window channel legitimately is.
22. **Active QQ messages are off by default.** They are rate-limited
   (certified 60 qpm / uncertified 30 qpm, 20 qpm per group, 1000/day per group)
   and pass through `MESSAGE_AUDIT`. Enabling them is a product decision, not a
   code default.
23. **QQ's `op=13` URL check has no signature.** The official sample sends only
   `User-Agent` and `X-Bot-Appid`, so the adapter skips signature verification
   for that op and proves identity through the *reply* signature instead.
   `X-Bot-Appid` is still compared against `app_id` when both exist.
24. **Full-mode QQ groups are stored, not answered.** `GROUP_MESSAGE_CREATE`
   (接收所有消息) has no "was I mentioned" field, so it is treated as
   un-addressed. Turn 全量模式 off, or set `answer_group_all` and accept that the
   bot may answer messages that never addressed it.
25. **QQ has no profile API and no QQ numbers.** Threads are keyed by
   `member_openid` (group) / `user_openid` (1:1) — an opaque string the console
   can only display as-is. `identity_bindings` therefore cannot merge a QQ
   customer with the same person on WeChat.
26. **Real QQ tenant is still unverified.** Everything above is offline-tested
   (58 cases) against the documented protocol; the callback URL, the `op=13`
   handshake and the 5-minute window have not been exercised against a live bot.
27. **QQ rich media is single-attachment.** `attachments` is a list, but only the
   first entry becomes `event.media`; a message with two images stores one and
   mentions the count in `raw`.

## Data model

Schema `omni_service` (created by `migrations/v1.0.0__init.sql`):

- `conversations` — unique on `(channel_kind, channel_account, external_user_id)`;
  carries `last_inbound_msg_id` + `reply_seq` for per-message reply quotas (QQ)
- `messages` — partial unique index on `(conversation_id, platform_msg_id)`
- `identity_bindings` — cross-channel natural-person merge
- `handoffs` — escalation ledger
- `webhook_seen` — replay idempotency

Media is **not** copied into a plugin folder: attachments land in the main
`public.media_files` + `admin/static/media`, and `messages.media_id` references
that row. The plugin can be uninstalled without orphaning customer assets.

## Architecture rules this plugin obeys

1. `plugins.chatbot.service.answer()` is the only dialogue engine. This plugin
   builds no prompts and calls no LLM directly.
2. `visitor_id` is always `channel_kind:external_user_id` (chatbot DEF-017).
3. Platform field names never leak above `channels/` — everything is `OmniEvent`.
4. Fail closed: missing credentials, bad signature, or corp mismatch → 403.
5. ACK fast, work off-thread: platform callbacks must not wait for an LLM turn.
6. Push vs pull is a channel property, not a pipeline branch: a channel
   implements `normalize()` (one event) or `pull()` (N events). The webhook
   handles both and never learns which platform it is talking to.
7. Only `actor=customer` events reach the AI. `actor=agent` means a human
   already replied inside the enterprise client, and `actor=system` means a
   platform notice — both are recorded (or dropped) without an AI turn.
8. Handshakes are channel business, not webhook business: `verify_url()` for
   GET-based ones (WeCom / 微信客服), `handshake()` for POST-based ones (Feishu
   challenge / QQ `op=13` signature), and neither ever reaches the pipeline.
9. Per-message reply quotas belong to the channel (`max_replies_per_inbound`,
   `window_blocks_reply`), but the slot is claimed inside the database
   (`orchestrator.next_reply_seq`), so no adapter ever does its own DB work.

## Tests

```bash
PY=python
$PY plugins/omni_service/tests/test_signing.py               # 13 cases, no DB
$PY plugins/omni_service/tests/test_normalize.py              # 28 cases, no DB
$PY plugins/omni_service/tests/test_orchestrator_contract.py  # 16 cases, no DB
$PY plugins/omni_service/tests/test_webhook_e2e.py            # 15 cases, no DB (needs Flask)
$PY plugins/omni_service/tests/test_window_and_ledger.py      # 10 cases, no DB
$PY plugins/omni_service/tests/test_wechat_kf.py              # 18 cases, no DB
$PY plugins/omni_service/tests/test_feishu.py                 # 36 cases, no DB (needs Flask for 3)
$PY plugins/omni_service/tests/test_dingtalk.py               # 43 cases, no DB (needs Flask for 2)
$PY plugins/omni_service/tests/test_qq.py                     # 58 cases, no DB (needs Flask for 4)
$PY plugins/omni_service/tests/test_pg_integration.py         # auto-skips without PG
```

Coverage boundary, stated honestly:

- `test_webhook_e2e` is the only suite that goes through the **real wire
  format** for WeCom — XML → AES-256-CBC → sha1 signature → HTTP → blueprint →
  `verify()` → `decrypt()` → `normalize()`. Everything below the pipeline entry
  is faked, so it proves the contract, not the SQL.
- `test_feishu` does the same for Feishu: the ciphertext is produced by
  `feishu_crypto.encrypt_for_test` (same IV-prefix + PKCS#7 shape the platform
  uses), signed with the real SHA256 construction, and pushed through the real
  blueprint, so signature and decryption are exercised, not mocked.
- `test_dingtalk` signs with the real HMAC-SHA256 construction and pushes through
  the real blueprint. There is no ciphertext layer to reproduce (DingTalk's robot
  callback is plaintext), so what it proves is the signature + 1h window + tenant
  binding + normalization — and it deliberately includes a test asserting that a
  **tampered body still verifies**, because pretending otherwise would hide
  limitation 13.
- `test_window_and_ledger` covers the expired-window branch that the
  orchestrator suite stubs out (that suite replaces `send_reply` wholesale).
- `test_qq` anchors on the **one official sample that is reproducible byte for
  byte**: the `op=13` answer signature (`87befc99…652c706`). The other demo on
  QQ's "验证签名" page has a body string that is mangled in the HTML and cannot
  be recomputed from any plausible spelling, so incoming-signature coverage uses
  round-trip cases instead, plus a case asserting that a `sha256`-derived
  signature is rejected. Its reply-slot cases run through the real
  `send_reply()` with the DB patched, which is the only place the 5-reply quota
  can be proved.
- SQL, migrations, `TIMESTAMPTZ` handling and concurrent DDL are covered only by
  `test_pg_integration`, which needs a real PostgreSQL. **It has not been run
  against the test host yet** — that remains an open verification item.
- No channel has been exercised against a **real tenant** yet. Signature and
  payload shapes are verified against the published contracts; the admin-console
  prerequisites above are still unverified in practice.

## API

All endpoints live under `/plugin/omni-service` and answer
`{ok, data, error, meta}`. Auth is fail-closed and two-channel
(`X-Internal-Token` or admin SSO).

```
GET  /channels
GET  /conversations?status=&channel=&assignee=&q=&limit=&offset=
GET  /conversations/<id>
GET  /conversations/<id>/messages?limit=&before=
POST /conversations/<id>/takeover
POST /conversations/<id>/release
POST /conversations/<id>/close
POST /conversations/<id>/read
POST /conversations/<id>/reply      {text}
GET  /stats
```

Webhook: `GET|POST /api/v1/omni/webhook/<channel_kind>`
