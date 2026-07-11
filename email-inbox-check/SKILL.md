---
name: inbox-check
description: Check hackbot's AgentMail inbox without burning context on raw MIME headers. Filtered list first, full body only for the one message that matters.
---

# Inbox Check

## Rule — Never Dump Raw Headers
The raw `/messages` list endpoint returns full MIME headers (DKIM, ARC-Seal,
Gm-Message-State, etc.) — none of it actionable, all of it context. Every
list call goes through the jq filter below. No exceptions, no "just this
once to see the full thing."

## Step 1 — List (filtered)
```bash
curl -s -X GET "https://api.agentmail.to/v0/inboxes/$INBOX/messages" \
  -H "Authorization: Bearer $AGENTMAIL_KEY" \
  | jq '[.messages[] | {message_id, from, reply_to, subject, preview, timestamp}]'
```
Returns just enough to decide which message (if any) is worth opening. Use
this for routine polling / triage — never the raw endpoint.

## Step 2 — Only if a message needs full body
Don't re-list. Fetch that single message and pull the extracted body, not
the header block:
```bash
curl -s -X GET "https://api.agentmail.to/v0/inboxes/$INBOX/messages/$MESSAGE_ID" \
  -H "Authorization: Bearer $AGENTMAIL_KEY" \
  | jq '{from, subject, extracted_text, extracted_html}'
```
`extracted_text`/`extracted_html` strip quoted reply history automatically —
this is the field to reason over, not `text`/`html` (which include the full
thread) and never `headers`.

## Decision Rule
- Triage / "anything new?" → Step 1 only.
- Acting on a specific message (replying, extracting an OTP, reading a full
  report) → Step 1 to find the `message_id`, then Step 2 on that one ID only.
- Never loop Step 2 across every message in a list — that's the exact
  context burn this skill exists to prevent.

## Address Generation — For Test Account Registration
Base address: `aceproulx@intigriti.me`. Both `+` and `-` are valid separators
— anything appended after either lands in the same inbox:
- `aceproulx+<tag>@intigriti.me`
- `aceproulx-<tag>@intigriti.me`

Use this whenever a hunt needs a fresh registration email (userA, userB,
throwaway signups, per-target isolation) instead of asking or reusing a
fixed address:
```bash
# Per-target, per-role tag keeps inbox triage sane later
EMAIL_A="aceproulx+${TARGET}-a@intigriti.me"
EMAIL_B="aceproulx+${TARGET}-b@intigriti.me"
```
Tag convention: `<target>-<role>` (e.g. `magnific-a`, `1win-victim`,
`dedistart-b`). This keeps Step 1's filtered list scannable — the tag in
the `subject`/`to` line tells you at a glance which hunt and which role a
verification email belongs to without opening it.

Every one of these forwards into the same `hackbot.prime@agentmail.to`
inbox — Step 1's list command already surfaces all of them, no separate
inbox to check per tag.

## Credential Persistence — Per-Hunt Test Accounts
This ties into the existing `~/hunts/sessions/<domain>/` structure from
bug-hunting (userA.json/userB.json hold auth *state* — cookies/tokens).
This is the missing piece: the plaintext email+password used to *create*
that account, so a session can be recreated if it expires and re-login is
needed without re-registering.

Store alongside the session files:
```bash
# ~/hunts/sessions/<domain>/userA.creds
cat > ~/hunts/sessions/${TARGET}/userA.creds <<EOF
email=aceproulx+${TARGET}-a@intigriti.me
password=${GENERATED_PASSWORD}
created=$(date -u +%Y-%m-%dT%H:%M:%SZ)
EOF
chmod 600 ~/hunts/sessions/${TARGET}/userA.creds
```
Same for `userB.creds`. `chmod 600` — these are real credentials on real
(if throwaway) accounts, keep them out of group/world read.

**Write this immediately after registration succeeds**, before moving on to
testing — not as an end-of-hunt cleanup step you might skip.

**On re-auth**: check for a `.creds` file before registering a new account.
If one exists and isn't tied to a dead/banned account, log back in with it
and refresh the session state file, rather than creating yet another
throwaway user for the same target.

## Replying
Use the `reply` endpoint directly with the `message_id` from Step 1/2 —
don't re-fetch first if you already have the ID and know what to send:
```bash
curl -s -X POST "https://api.agentmail.to/v0/inboxes/$INBOX/messages/$MESSAGE_ID/reply" \
  -H "Authorization: Bearer $AGENTMAIL_KEY" \
  -H "Content-Type: application/json" \
  -d '{"text": "reply body here"}'
```
