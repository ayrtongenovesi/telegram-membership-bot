# Architecture & Data Model

## High-level

The system is a **stateless orchestration layer (n8n)** sitting between three services:

- **Telegram Bot API** — user-facing interface and side-effects (invite links, kicks)
- **Google Sheets** — single source of truth for user state
- **Gemini API** — natural-language understanding for FAQ

All business logic lives in n8n workflows. No code deployment outside n8n itself.

## Data model

Single Google Sheet, one tab, one row per user.

| Column | Type | Notes |
|---|---|---|
| `user_id` | integer | Telegram numeric ID. Primary key. |
| `username` | string | Telegram `@handle` (may be empty). |
| `first_name` | string | From Telegram profile. |
| `last_name` | string | From Telegram profile (may be empty). |
| `fecha_ingreso` | ISO date | When `/start` was first received. Drives all trial math. |
| `dias_en_grupo` | computed | `Math.floor(now - fecha_ingreso)` in days. Not stored, recomputed at read time. |
| `estado` | enum | `active_trial` · `trial_vencido` · `membresia_activa` |
| `fecha_pago` | ISO date | Set by `/pago` admin command. Empty for trial users. |
| `fecha_vencimiento` | ISO date | `fecha_pago + 30d`. Used for renewal cycle. |
| `avisado_15` | boolean | Day-15 reminder flag (idempotency). |
| `avisado_27` | boolean | Day-27 reminder flag (idempotency). |

### State transitions

```
                    /start                    day 30
   (no row)  ───────────────►  active_trial ─────────►  trial_vencido
                                    │
                                    │ /pago @user
                                    ▼
                              membresia_activa
                                    │
                                    │ fecha_vencimiento reached
                                    ▼
                                  (admin renews or status flips to expired)
```

## Workflows

### 1. Onboarding & FAQ
**Trigger:** `Telegram On Message`

```
On Message
  → Get Rows (filter by user_id)
  → IF (row exists)
      ├─ TRUE  → Switch on message text
      │            ├─ matches "/pago @x"  → admin allowlist check → update target row
      │            ├─ matches tutorial keywords → send static tutorial
      │            └─ default → Gemini (with FAQ context) → reply
      └─ FALSE → Append row (active_trial, fecha_ingreso=now)
                → createChatInviteLink (member_limit=1)
                → Send welcome + inline button with invite link
```

### 2. Trial Reminders
**Trigger:** `Schedule` (daily, 9:00)

```
Schedule
  → Get Rows (all users)
  → Loop:
      compute days = floor(now - fecha_ingreso)
      → IF (days == 15 AND avisado_15 != true AND estado == active_trial)
          → Send day-15 message
          → Update row: avisado_15 = true
      → IF (days == 27 AND avisado_27 != true AND estado == active_trial)
          → Send day-27 message
          → Update row: avisado_27 = true
```

### 3. Trial Expiration
**Trigger:** `Schedule` (daily)

```
Schedule
  → Get Rows
  → Filter: days_since_ingreso >= 30 AND estado == active_trial
  → For each:
      → Telegram kickChatMember (premium channel)
      → Send goodbye message
      → Update row: estado = trial_vencido
```

## n8n node settings worth knowing

- **Parse Mode: HTML** on every `Telegram Send` node. Markdown's escape rules break on user-generated content.
- **Append n8n Attribution: OFF** on every Send node, or every message ends with the n8n promo footer.
- **Get Rows → Always Output Data: TRUE**. Otherwise the IF node downstream sees nothing and the branch silently fails. Filter inside the IF instead of relying on the read returning zero rows.
- **`createChatInviteLink` with `member_limit: 1`** — each link is single-use, tied to that user record.

## Days calculation (trial math)

```js
Math.floor(
  $now.diff(
    DateTime.fromISO($json.fecha_ingreso),
    'days'
  ).days
)
```

`$now` and `DateTime` are from Luxon, available inside n8n Function/Code nodes.

## Why idempotency via sheet flags

The daily schedule could run twice (n8n restart, manual re-trigger, clock drift). Without flags, a user could get the same reminder twice. With `avisado_15` and `avisado_27` as boolean columns checked in the `IF` upstream of the Send, replays are safe. The downside is two extra columns in the sheet — worth it.

## Security

- Admin commands (`/pago`) gated by a hardcoded allowlist of Telegram user IDs inside the workflow. Trade-off: rotating an admin = editing the workflow. Acceptable for a 1-admin operation; would move to a `roles` tab in the sheet for multi-admin setups.
- Single-use invite links prevent leak-and-share.
- Bot does not give financial advice, does not promise returns, does not recommend position sizes. FAQ scope is enforced via the Gemini system prompt.

## What I'd change for v2

- Move the user table from Sheets to Postgres once row count > a few hundred. Sheets read-latency starts dominating.
- Add a `payments` log table separate from `users`. Right now `fecha_pago` only stores the most recent payment.
- Webhook receiver for an actual payment provider instead of the manual `/pago` step. Removes the admin from the critical path.
- Surface metrics (active trials, conversion rate, churn) in a tiny dashboard. Today the only way to see them is to look at the sheet.
