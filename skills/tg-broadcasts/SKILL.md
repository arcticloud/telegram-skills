---
name: tg-broadcasts
description: "Use when a Telegram bot sends the same or scheduled messages to many users: mailings, follow-up chains, reminders, re-engagement, launching a send to an existing base. Covers rate limits, `429` and `retry_after`, paid broadcasts, consent checks, unsubscribe, duplicate protection, blocked users and safe restarts. Triggers: \"рассылка в боте\", \"дожимы\", \"mass message\", \"broadcast to users\", \"message all subscribers\", \"too many requests\". Use tg-miniapp-delivery for one buyer's failed delivery."
license: MIT
---

# Bot broadcasts

A broadcast is a job that can stop at any point. It must be safe to restart, must not message anyone twice, and must stop
for a person the moment they opt out.

## Limits

From the Telegram Bots FAQ:

| Scope | Limit |
|---|---|
| One chat | About 1 message per second; short bursts may pass, then `429` |
| One group | 20 messages per minute |
| Bulk, free | About 30 messages per second |
| Bulk, paid broadcasts | Up to 1000 messages per second, 0.1 Stars for each message above the free 30 per second |

Paid broadcasts are enabled in @BotFather and sent with `allow_paid_broadcast`. Only successful sends are charged. Check the
current numbers before relying on them.

These are ceilings, not targets. A cold base also needs a slow start: a cap per run spreads the first wave over days, keeps
replies manageable, and limits the damage from a bad text or link.

## The send loop

For each person:

1. **Check eligibility right before the send**, not when the queue was built: consent, unsubscribe, already bought, blocked.
   A person can opt out while the queue is running.
2. **Render first.** If a placeholder has no value, skip and log it. Never send a message with a hole in it.
3. **Claim the slot in the database** with a unique key for person and message, status `sending`. If the claim fails,
   someone else has it.
4. **Send.**
5. **Mark `sent`.**

Handle the result:

| Result | Action |
|---|---|
| Success | Mark `sent`, pause, next person |
| `403` | Mark as blocked, keep it out of later runs, next person |
| `400 chat not found` | Observed when the person never started the bot. Release the claim, skip this run, do not count it as a block |
| `429` | Not sent: release the claim, stop this pass and wait `parameters.retry_after`. Do not push on with the queue |
| Timeout, `5xx` | Unknown outcome: the message may have arrived. Keep the claim for review, or accept that a retry can duplicate it |
| Other `4xx` | Definite failure: release the claim and let a later run retry, with a limit |

A `sending` row left after a crash is also an unknown outcome. Decide up front whether a duplicate or a missed message is
worse for this mailing, and write that down. For an advertisement, prefer silence. For a payment or access message, do
not resend blindly: hold it for review, or word it so that reading it twice does no harm.

## Consent and opt-out

- Promotional messages need a separate recorded consent. Service messages about a purchase are a different category.
- Put an unsubscribe button on every promotional message. It must take effect on the next check, without a restart.
- Keep service messages working after an opt-out from promotions.

## Before a real send

- Send every message to yourself with real values, media and buttons. A test mode that substitutes values catches most
  mistakes.
- Check that each button leads where the message says, from a phone.
- Count who will receive it, and exclude the owner and test accounts.
- Know how to stop it: one switch, no deployment.

## Restarts and deployment

- Keep the queue in the database, not in memory. A restart resumes from the rows, not from the top.
- Run one sender. Use a lock or a single scheduled job, or two instances will race for the same people.
- Let the current send finish before stopping the process.
- Do not ship a broadcast by accident: if the sender starts on deployment, keep it behind an explicit switch.

## Reporting

Report sent, skipped with reason, blocked and failed. For a claim of "sent", read the stored status or the Telegram
response. A message visible in one client is not proof that the job finished.
