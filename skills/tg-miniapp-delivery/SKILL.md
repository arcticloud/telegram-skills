---
name: tg-miniapp-delivery
description: "Use when a Telegram Mini App takes payments or grants access and the bot must then reach the buyer: paying without pressing Start, `400 Bad Request: chat not found`, `requestWriteAccess`, `allows_write_to_pm`, `write_access_allowed`, failed delivery after payment, retrying delivery, owner alerts about a paid order. Triggers: \"paid but the bot cannot message them\", \"оплата мимо бота\", \"не запускал бота\", \"chat not found\", \"выдача не завершена\", \"бот спамит предупреждением\". Use tg-broadcasts for bulk sends."
license: MIT
---

# Mini App delivery

A Mini App and the bot chat are separate surfaces. A person can open the Mini App by direct link, pay, and never press
Start. The Mini App still works, because it authenticates with `initData`. The bot may still be unable to write first.

Observed on a live bot: the user row existed, the payment succeeded, and every `sendMessage` failed with
`400 Bad Request: chat not found`. The buyer was not blocked and had done nothing wrong. A delivery step that treated the
message as the moment of purchase kept failing, and a retry loop sent the owner the same alert every 5 minutes.

## Rules

1. **Entitlement comes from the payment record.** Compute access from the stored purchase, never from a delivered message.
   A chat message is a copy of what the Mini App already shows.
2. **Ask for write access in the Mini App.** Do it after the purchase, or right before a step that depends on bot messages.
3. **Treat failed delivery as a state.** Keep it, retry it silently, and tell the owner once.
4. **Give the owner something to act on.** Use the order number they already see, a name and @username. A bare Telegram id
   is useless to a person.

## Asking for write access

`Telegram.WebApp.requestWriteAccess([callback])` (Bot API 6.9+) shows a native popup. The callback gets a boolean.

```js
function askWriteAccess(tg, isBuyer) {
  if (!tg || !isBuyer) return;
  if (!tg.requestWriteAccess || !tg.isVersionAtLeast || !tg.isVersionAtLeast("6.9")) return;
  var user = tg.initDataUnsafe && tg.initDataUnsafe.user;
  if (user && user.allows_write_to_pm) return;
  var asked = false;
  try { asked = localStorage.getItem("writeAsked") === "1"; } catch (e) { /* closed storage: still ask */ }
  if (asked) return;
  try {
    tg.requestWriteAccess();
    try { localStorage.setItem("writeAsked", "1"); } catch (e) { /* may ask again next time */ }
  } catch (e) { /* the request failed: ask next time, the app keeps working */ }
}
```

- Render the screen first. The prompt is an addition, never a gate on content the person paid for.
- `initDataUnsafe` is fine for this UI decision. It is not proof of anything, so never use it for access or payment
  decisions. Validate `initData` on the server for those.
- `allows_write_to_pm` is optional in `WebAppUser`. Its absence does not prove the bot cannot write, so a person who already
  started the bot may see the prompt once. The documentation does not promise that the popup is skipped when access exists.
- A declined prompt is an answer. Ask once per device, then keep the normal Mini App path working.

After a person accepts, the bot receives a service message with `write_access_allowed`, and `from_request` is true.
Handle it to deliver at once. A periodic retry also works, with a delay.

## Reading send errors

| Result | Meaning | Action |
|---|---|---|
| `400 chat not found` | Observed when the person never started the bot; Telegram does not document this as the only cause | Keep the record, retry later, ask for write access in the Mini App |
| `403` | The bot was blocked or cannot write to this user | Mark as blocked for bulk sends; a later unblock can be retried |
| `429` | Too many requests | Wait `parameters.retry_after`, do not count it as a delivery failure |
| Timeout, `5xx` | Unknown outcome: the message may have arrived | Do not assume failure. Prefer an idempotent next step over a blind resend |

Error text comes from the server and is not a stable contract. Log the original description next to your own wording.

## Retrying delivery

- Give each visible step its own checkpoint, set right after that step succeeded. With one flag for several steps, a retry
  after a partial success sends the first message again.
- Guard concurrent runs: a webhook, a poll and a "check payment" button can all fire at once for the same purchase.
- Limit the retry window, and leave access in place when it ends.
- Keep everything after the delivered flag out of the retry path, or it repeats too.

## Alerting the owner

- **One alert per purchase, not one per retry.** Mark it as alerted only after the alert was accepted by Telegram. If you
  mark it first, a network failure loses the only alert. Keep the buyer notice on a separate flag for the same reason.
- **Close the loop.** Send one line when delivery finally succeeds, or silence looks the same as a stuck order.
- **In-memory flags reset on restart.** That gives one repeat alert per restart. Store them if that is too noisy.
- **The alert path must not throw.** Name lookups and link bindings read the database. If they fail inside a `catch`, the
  alert and the buyer notice are both lost. Wrap them and keep a fallback text that needs no database.
- **Say what to do.** State the reason in plain words and whether access is already open. Do not point to a screen that
  does not show this order.

```text
⚠️ Paid, access message not delivered
💰 No. 19 · 2,990 ₽ · Guide · Anna @anna · 6 Nov, 15:39
Reason: this person has not started the bot. Access in the Mini App is already open.
The bot keeps retrying silently for 24 hours and will report when it arrives.
```

Without a username, show the email from the payment and a `tg://user?id=` link. That link may not open if the client does
not know this user, so keep the email.

## One order number

Use one human-facing number everywhere: alerts, CRM, sales feed and owner commands. A database id next to a separate order
number looks like two counts, and it is dangerous: `/refund #19` can open another person's purchase. If test purchases have
no order number, give them a visibly different form such as `p23`, and show the buyer in every confirmation.

## Verify

Green tests do not prove this path. Check it with an account that has never started the bot:

1. Open the Mini App by direct link and pay, or use a test purchase.
2. Confirm that access opens while the first send fails.
3. Confirm the owner gets one alert with a name and an order number.
4. Accept the prompt and confirm the message arrives and the closing line is sent.

A user who started the bot and then blocked it gets `403`, so that account does not test this case.
