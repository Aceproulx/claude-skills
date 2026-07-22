---
name: mobile-hacking
description: Android app pentesting on non-rooted devices. ADB for screen state/screenshots, Objection for FLAG_SECURE bypass, Frida Gadget for runtime hooking, Caido MCP for traffic, jadx MCP for static analysis.
---

# Mobile Hacking (Android, Non-Rooted)

## AUTONOMOUS MODE — DO NOT ASK THE USER
Make every decision yourself. Same rule as bug-hunting: no permission-seeking,
no confirmation loops. Log dead ends to `interesting.md`, pivot after 10
failed attempts on a given approach.

## Environment Assumptions
- **Non-rooted device** running the target APK repackaged with Frida Gadget
  (objection's `patchapk` or manual gadget injection).
- **Objection** is the primary tool for anything that would normally need
  root — UI security bypasses, SSL pinning disable, keystore dumps.
- **Frida server model**: Gadget, not frida-server. You attach to the
  already-injected Gadget, you don't spawn/inject yourself.

## Session Persistence
- **Session directory**: `~/hunts/sessions/mobile-<app-package>/`
- Store: `device.info` (adb device id, Android version), `hook-notes.md`
  (which hooks fired, which sinks matter), `screenshots/` (timestamped).
- If a repackaged APK + Gadget config already exists for this app, reuse it —
  don't re-patch unless the app was updated.

## ADB — Screen State & Screenshots
Use ADB as the eyes of the hunt. Before touching Frida/objection, confirm
device state:

```bash
adb devices                                  # confirm device connected
adb shell wm size                            # screen dimensions, for coord taps
adb exec-out screencap -p > screenshots/$(date +%s).png
```

- Take a screenshot after every meaningful UI action (login, nav to a
  sensitive screen, triggering a permission prompt) — this is your evidence
  trail for the writeup, same role agent-browser screenshots play in web
  hunts.
- If `screencap` returns a **black/blank image**, the activity has
  `FLAG_SECURE` set (common on banking/payment/OTP screens) — this is the
  trigger to bypass it, not a dead end.

## ADB — UI Interaction via Coordinates
No accessibility/automation framework needed — drive the app the same way a
finger would, using raw taps/swipes. This is how you navigate to a screen,
trigger a flow, or mash through a UI state without objection or Frida
involved at all.

```bash
adb shell input tap <x> <y>                       # single tap
adb shell input swipe <x1> <y1> <x2> <y2> <ms>    # swipe/scroll
adb shell input text "some_string"                 # type into a focused field
adb shell input keyevent 66                         # ENTER
adb shell input keyevent 4                          # BACK
adb shell input keyevent 3                          # HOME
```

- **Finding coordinates**: take a screenshot (`adb exec-out screencap -p`),
  open it, read pixel coords off the image directly — screen dimensions from
  `adb shell wm size` give you the bounds to work within.
- **More precise than eyeballing pixels**: dump the UI hierarchy instead of
  guessing from a screenshot —
  ```bash
  adb shell uiautomator dump /sdcard/ui.xml
  adb pull /sdcard/ui.xml
  ```
  Parse the XML for the target element's `bounds="[x1,y1][x2,y2]"` and tap
  the center point. Far more reliable than pixel-guessing off a screenshot,
  especially on scaled/DPI-weird devices — use this as the default method,
  fall back to screenshot-coordinate-guessing only if a screen refuses to
  dump (some FLAG_SECURE screens block uiautomator too — bypass via
  objection first, same as the screenshot case).
- **Scripted flows**: chain tap/swipe/text sequences into a shell script per
  app flow (login, checkout, settings nav) so repeated testing runs (e.g.
  re-triggering a race condition or re-testing after a hook change) don't
  require manual re-navigation every time.
- **Combine with screenshots**: tap → sleep 1s → screenshot → confirm state
  before the next tap, especially on flows with loading states or animations
  that make blind chained taps unreliable.
- **Text fields**: `input text` doesn't handle spaces well on some Android
  versions — use `%s` in place of spaces, or tap the field then use
  `adb shell input keyevent` sequences for special chars if `%s` fails.

## FLAG_SECURE Bypass — Objection (Non-Root Path)
Since there's no root, don't try `wm` tricks or a rooted screenshot method —
objection's runtime patch is the correct tool here.

```bash
objection -g <package.name> explore
# inside objection:
android ui FLAG_SECURE false
```

- This flips the flag at runtime via Frida under the hood — no APK repatch
  needed if Gadget is already attached.
- Re-run `adb exec-out screencap -p` immediately after — confirm the capture
  is no longer black before moving on.
- Log which screen had FLAG_SECURE set in `hook-notes.md`. A screen that
  actively hides itself from screenshots is a spidy-sense signal on its own —
  it usually means sensitive data (card numbers, OTP, seed phrases) renders
  there. Pivot into that screen's traffic once the bypass confirms.
- Other objection one-liners worth running early on any new target:
  ```bash
  android sslpinning disable          # before touching Caido — no pinning, no MITM
  android keystore list               # enumerate stored keys/certs
  android hooking list activities     # map attack surface
  android hooking list classes        # for later frida script targeting
  ```

## Frida — Runtime Hooking
Gadget model — attach, don't spawn:

```bash
frida -H 127.0.0.1:27042 Gadget -l ~/hunts/mobile-tools/apks/lab/hook.js
```

- `-H 127.0.0.1:27042` is the Gadget's exposed port (set via the Gadget
  config when the APK was patched) — confirm this matches the current
  session's patched build before assuming a hook failure is a script bug.
- Hook script targeting comes from `android hooking list classes` /
  `list class-methods` output in objection — build `hook.js` from confirmed
  class/method names, not guesses.
- **What to hook, feature-driven** (same philosophy as web — the feature
  determines the attack, not a checklist):
  - Crypto calls (`javax.crypto.Cipher`, custom crypto wrappers) — weak
    keys, hardcoded IVs, ECB mode.
  - Root/jailbreak/emulator detection methods — bypass to keep testing on
    the lab device.
  - Certificate pinning implementations objection's blanket bypass missed
    (custom `TrustManager`/`HostnameVerifier` classes — check
    `list classes` for anything not matching known pinning libs).
  - Local auth checks (biometric/PIN gate methods) — return true to skip if
    it's blocking further exploration.
  - Any method touching `SharedPreferences`, keystore, or local DB — log
    args/return values to spot plaintext secrets.
- Log every hook that fires with meaningful data to `hook-notes.md`
  immediately — this is the mobile equivalent of `interesting.md`, same
  discipline: log even if it looks like a dead end.

## jadx MCP — Static Analysis
Use jadx MCP to read decompiled source rather than eyeballing raw smali:

- Pull the APK's class list and grep for interesting sink patterns before
  writing Frida hooks — confirms real method signatures instead of guessing
  from objection's runtime class list alone.
- Cross-reference: if Frida hooking shows a method firing with suspicious
  data, pull that method's decompiled source via jadx MCP to understand the
  full logic path (what calls it, what it does with the return value)
  before deciding if it's actually exploitable.
- Look for: hardcoded API keys/secrets, debug flags left in release builds,
  exported components (`Activity`/`Service`/`Receiver`/`Provider` with
  `exported="true"` in the manifest — pull `AndroidManifest.xml` via jadx
  MCP first, it's the fastest map of attack surface on a new APK).
- Deep-link handling code is a priority read — intent-based attacks
  (unvalidated deep link params, implicit intent hijacking) are a common
  class jadx surfaces fast that dynamic testing alone can miss.

## Caido MCP — Traffic Interception
Once SSL pinning is down (via objection above), route the device's traffic
through Caido same as any web target:

```bash
adb shell settings put global http_proxy <caido-host>:8080
```

- Install Caido's CA cert on the device (`adb push` the cert, then install
  via device settings, or objection's cert-pinning bypass path if pinning
  fights back after the proxy is set).
- From here, standard bug-hunting skill vulnerability classes apply to the
  mobile app's API traffic: IDOR, mass assignment, JWT attacks, auth
  bypass — same curl/Caido workflow as web, the client is just the app
  instead of a browser.
- Correlate: an interesting Frida hook (e.g. a local auth check) that also
  corresponds to an API call in Caido history is a strong lead — client-side
  gate + server-side trust in the client is a classic bypass chain.

## Vulnerability Classes — Mobile-Specific
- **Insecure local storage**: SharedPrefs, SQLite DBs, files in
  `/data/data/<package>/` — pull via objection (`android sms list` /
  browse commands) or hook read/write calls directly with Frida.
- **Exported component abuse**: launch exported activities directly via
  adb, bypassing intended app flow:
  ```bash
  adb shell am start -n <package>/<exported.activity>
  ```
- **Deep link / intent injection**: craft malicious intents targeting
  exported components found via jadx manifest read.
- **Weak crypto**: confirmed via Frida hooks on `Cipher`/`MessageDigest`
  calls — hardcoded keys, ECB, weak hashing for sensitive fields.
- **Client-side trust**: any check (root detection, jailbreak, local auth,
  license/subscription check) that's client-side only — confirm by hooking
  it, forcing a bypass, and checking whether the server ever validates the
  same thing independently.
- **WebView issues**: JS bridge exposure (`addJavascriptInterface`),
  `setJavaScriptEnabled` + loading untrusted URLs, file:// access — read
  WebView setup code via jadx MCP.

## Validation Requirement
Same discipline as web: before logging a finding as confirmed, capture raw
evidence — screenshot (post-FLAG_SECURE-bypass if relevant), Frida hook
output, and the corresponding Caido request ID. Don't log conclusions,
log evidence, and let @bug-validator (or manual re-check) confirm.
