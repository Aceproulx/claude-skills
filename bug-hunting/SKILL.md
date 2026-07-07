---
name: bug-hunting
description: Main-app focused bug hunting. Pick one feature, pivot deep. Use Caido Match & Replace for UI bypass. No wide recon, no subdomain hunting.
---

# Bug Hunting

## AUTONOMOUS MODE — DO NOT ASK THE USER
Make every decision yourself. Never ask for permission, clarification, or confirmation. If stuck, pivot to a different attack class. If nothing works for 10 attempts, log it and move on.

## Strategy: Main App Guy
- **One target**: the main application. Not subdomains, not staging, not api. subdomain.
- **One feature**: pick one core feature (search, profile, checkout, upload, etc.) and go deep on it
- **Pivot hard**: every response is a lead. A 403 means test auth bypass. A verbose error means probe for injection. A user ID in a response means test IDOR.
- **No wide hunting**: don't waste tokens enumerating URLs, subdomains, or sourcemaps. You already have the target. Hack it.

## UI Bypass via Caido Match & Replace
Use Caido's `create_tamper_rule` / `toggle_tamper_rule` to modify traffic in transit — this lets you bypass frontend restrictions without touching the browser:

Common tamper rules to create:
- **Premium bypass**: match `"isPremium":false` → replace `"isPremium":true`
- **Role escalation**: match `"role":"user"` → replace `"role":"admin"`
- **Feature flags**: match `"featureEnabled":false` → replace `"featureEnabled":true`
- **Paywalls**: match `"subscriptionTier":"free"` → replace `"subscriptionTier":"enterprise"`
- **Rate limits**: match `X-RateLimit-Remaining:\s*0` → replace `X-RateLimit-Remaining: 999`
- **Disabled buttons/inputs**: match `"disabled":true` → replace `"disabled":false`
- **Hidden fields**: match `"hidden":true` → replace `"hidden":false`
- **Any UI gating**: look for boolean/flag responses that gate UI features and flip them

Steps:
1. Browse the feature in agent-browser, capture the request in Caido
2. Inspect the response for boolean flags, roles, tiers, feature gates
3. Create a tamper rule in Caido to flip the flag on subsequent requests
4. Reload the page in agent-browser to see the unlocked UI
5. If something new appears, that's an attack surface — pivot into it

## Attack Methodology (per feature)
1. Use agent-browser to interact with the feature while Caido captures traffic
2. Inspect Caido history for API calls the feature makes
3. For each endpoint, test: IDOR (change IDs), auth bypass (remove/send wrong cookies), mass assignment (add extra fields), injection (mess with params)
4. If any response shows flags, roles, limits, tiers — create tamper rules and reload
5. If the unlocked UI reveals new endpoints, repeat from step 1 on those

## Fuzzing
- **Payloads**: stored at `/mnt/d/Desktop/coffinxp-payloads`
- **Path fuzzing**: use `Pentester_wordlist.pay` from that directory
- **If site is too restrictive**: skip fuzzing entirely. Concentrate on finding vulns directly through manual testing, tamper rules, and logic analysis. No point fighting WAFs/rate limits — pivot to smarter attacks.

## Vulnerability Classes (in priority order)
1. **IDOR** — change numeric/UUID IDs in requests. Check responses for other users' data.
2. **Auth bypass** — remove cookie, change cookie, use another account's cookie. Check if endpoints actually check auth.
3. **Mass assignment** — add extra fields to JSON bodies. Try `role`, `isAdmin`, `premium`, `featureFlags`.
4. **XSS** — inject into every reflected parameter. Check DOM sinks in JS responses.
5. **SSRF** — any URL parameter, webhook, file fetch, image URL. Point at internal targets.
6. **Injections** — SQLi (parameter pollution), template injection, command injection in file operations.

## Tools

### Sending HTTP Requests — CAIDO ONLY
**Never use curl, python, or any tool other than Caido.**
- `send_request` / `edit_request` / `batch_send` for all HTTP traffic
- `list_requests` with HTTPQL to find requests in history
- `create_tamper_rule` / `toggle_tamper_rule` for Match & Replace UI bypass
- Everything stays in Caido history for later inspection

### Browser Testing — AGENT-BROWSER ONLY
- Always `--headed` for XSS validation
- Workflow: open → snapshot -i → interact via refs → re-snapshot

## Validation Requirement
Before logging any finding as confirmed, invoke the bug-validator skill. Do not mark CONFIRMED yourself — only the validator's verdict counts.
