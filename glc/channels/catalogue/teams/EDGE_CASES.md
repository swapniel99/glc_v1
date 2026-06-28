# Microsoft Teams adapter — wire-format edge cases

> Companion reference to [`README.md`](./README.md). The README covers
> setup and the happy path; this file documents the edge cases the
> adapter handles, the wire-format payloads that trigger each one, and
> the source-of-truth in [`adapter.py`](./adapter.py).
>
> Audience: the GLC v1 maintainer reviewing PR #12 on
> `theschoolofai/glc_v1`, and future contributors extending the
> adapter (real-tenant deployment, additional Bot Framework activity
> types, message-extensions).

## TL;DR

The Bot Framework Activity protocol is intentionally broad — the same
HTTP webhook receives real user messages, channel housekeeping events
(joins, typing), interactive card submissions, and reconnection
signals all under a single envelope shape. An adapter that only
handles the obvious `type == "message"` case loses user intent on
the non-obvious paths. The four cases below are the ones that
mattered when porting our channel onto the GLC `ChannelMessage`
envelope.

## 1. Non-message activity types — silently ignored

**Wire format.** The Bot Framework Connector and the Bot Framework
Emulator both emit non-message activities on the same webhook as
user messages. The activity types we observed during the emulator
demo:

```json
{ "type": "conversationUpdate", "membersAdded": [...] }
{ "type": "typing", "from": {...} }
{ "type": "endOfConversation", "code": "completedSuccessfully" }
```

**Why it matters.** If `on_message` returned a `ChannelMessage` for
each of these, the gateway would trigger an agent run on every
keystroke (typing) and every channel-join (conversationUpdate),
flooding the LLM cost log with no-op invocations.

**Where it's handled.**
[`adapter.py:147`](./adapter.py) — `if activity.get("type") != "message": return None`.

**Reference.** [Bot Framework Activity types](https://learn.microsoft.com/en-us/azure/bot-service/rest-api/bot-framework-rest-connector-activities#activity-types).

## 2. `serviceUrl` is dynamic per conversation

**Wire format.** Every inbound Activity carries a `serviceUrl` field
that the Connector uses to route a reply back to that specific
conversation. The URL changes per region, per tenant, per conversation
restart:

```json
{
  "serviceUrl": "https://smba.trafficmanager.net/in/",     // India tenant
  "serviceUrl": "https://smba.trafficmanager.net/amer/",   // Americas tenant
  "serviceUrl": "http://localhost:60550/",                  // Bot Framework Emulator
  ...
}
```

**Why it matters.** Hard-coding `serviceUrl` works for the demo and
breaks immediately on real-tenant deployment. The `ChannelReply`
envelope is deliberately channel-agnostic and does not carry it, so
the adapter must remember which URL goes with which sender.

**Where it's handled.**
[`adapter.py:205`](./adapter.py) — `self._contexts[from_id] = {"service_url": ..., "conversation_id": ...}` on every inbound message; looked up again at
[`adapter.py:237`](./adapter.py) when constructing the outbound POST URL.

**Failure mode.** If `send()` is called for a sender we have not
observed via `on_message`, the adapter raises `RuntimeError` rather
than silently dropping the reply or guessing the URL.

## 3. Adaptive Cards arrive as attachments with no `text`

**Wire format.** When a user submits an Adaptive Card form (or taps
a Card button that triggers a message), the activity carries a
`contentType: "application/vnd.microsoft.card.adaptive"` attachment
and **no top-level `text` field**. The user's intent is encoded in
the card's `body[]`:

```json
{
  "type": "message",
  "from": {...},
  "text": "",
  "attachments": [{
    "contentType": "application/vnd.microsoft.card.adaptive",
    "content": {
      "type": "AdaptiveCard",
      "body": [
        { "type": "Container", "items": [
          { "type": "TextBlock", "text": "Approve PR #12" }
        ]}
      ]
    }
  }]
}
```

**Why it matters.** An adapter that reads only `activity.text` returns
an empty `ChannelMessage.text` to the gateway, and the agent sees
"the user sent nothing." The user's actual intent — "Approve PR #12" —
is lost.

**Why a breadth-first walk.** Real cards in production routinely nest
TextBlocks inside `Container`, `ColumnSet`, or `Column` containers
(see the Adaptive Cards [layout reference](https://adaptivecards.io/explorer/Container.html)). A naive
`body[0].text` read misses these. The adapter walks the body
breadth-first.

**Where it's handled.**
[`adapter.py:50`](./adapter.py) — `_extract_adaptive_card_text`
recursively walks `body`, descending into `items` and `columns`.
[`adapter.py:187`](./adapter.py) — the extracted text is promoted to
`ChannelMessage.text`, and the raw card JSON is preserved under
`metadata["adaptive_card"]` for downstream code that wants the
structured form (e.g. re-rendering a follow-up card).

## 4. Tenant binding for the OAuth token

**Wire format.** The outbound reply path requires a Bot Framework
access token, obtained via OAuth client-credentials. Microsoft
deprecated the multi-tenant `botframework.com` token endpoint after
2025-07-31; new bot registrations must use the single-tenant Azure AD
endpoint:

```
POST https://login.microsoftonline.com/{TEAMS_TENANT_ID}/oauth2/v2.0/token
  grant_type=client_credentials
  client_id={TEAMS_APP_ID}
  client_secret={TEAMS_APP_PASSWORD}
  scope=https://api.botframework.com/.default
```

**Why it matters.** Reverting to the deprecated endpoint compiles and
runs locally but silently fails against real Teams tenants
post-2025-07-31. The error is a 400 with a generic
`unauthorized_client` payload that doesn't mention the deprecation.

**Where it's handled.**
[`adapter.py:92`](./adapter.py) — `_get_bot_framework_token` targets
the single-tenant endpoint and caches the token keyed by `TEAMS_APP_ID`
with a 60-second refresh margin against the `expires_in` returned by
Azure AD.

**Verify before relying on it.** The CI suite exercises the mock
path only. The real OAuth path was not tested against a live
Microsoft 365 tenant — the Microsoft 365 Developer Program tightened
its policy in late 2025 and the team could not obtain a free test
tenant during the assignment window. Verify against current
[Microsoft Bot Framework auth docs](https://learn.microsoft.com/en-us/azure/bot-service/rest-api/bot-framework-rest-connector-authentication)
before a production deployment.

## Behaviour table — quick reference

| Inbound shape | `on_message` returns | `ChannelMessage.text` | `metadata` keys |
|---|---|---|---|
| `type=message`, plain `text` | `ChannelMessage` | the `text` field | `service_url`, `conversation_id`, `tenant_id` |
| `type=message`, Adaptive Card attachment | `ChannelMessage` | first TextBlock body, walked breadth-first | + `adaptive_card` (raw JSON) |
| `type=message`, image/file attachment | `ChannelMessage` | the `text` field (may be empty) | + `attachments[]` of `Attachment(kind, ref, mime)` |
| `type=conversationUpdate` | `None` | — | — |
| `type=typing` | `None` | — | — |
| `type=endOfConversation` | `None` | — | — |
| Forced HTTP/WS disconnect mid-session | `None` (no raise) | — | — |
| Public channel + sender not on allowlist + no @mention | `None` (silently dropped) | — | — |

## Trust-level mapping

| Inbound source | `trust_level` set by `classify()` |
|---|---|
| Paired owner identity (see `setup/trust_setup.py --owner`) | `owner_paired` |
| Authenticated user, paired (`setup/trust_setup.py --user`) | `user_paired` |
| Unknown sender / `setup/trust_setup.py --revoke` | `untrusted` |

The trust level is the **first** field the policy engine inspects on
the envelope; the adapter never reaches into the policy engine
itself. See [`glc/security/trust_level.py`](../../../security/trust_level.py)
for the classifier and `policy.yaml` at the repo root for the rules.

## Acknowledgements

Adapter, schemas, demo, and emulator runner — Sudip Patil
([@spatil1985](https://github.com/spatil1985)), with credentials and
Azure configuration support from Swapnil Gusani
([@swapniel99](https://github.com/swapniel99)) and CI workflow
fixes from raghav venkat. Edge-case reference assembled by Abhinav
Rana ([@levelscorner](https://github.com/levelscorner)) from the
existing adapter source.

Full Group Teams (Group 15) roster per the EAG V3 course LMS:

- Abhilash Manikanta
- Abhinav Rana
- Ankita Sinha
- Balaji Chunduri
- Kevin S
- raghav venkat
- Ravi Shankar
- sai kiran
- Sudip Patil
- Swapnil Gusani
- Vasanth Balamurugan
