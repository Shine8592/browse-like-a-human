---
name: browse-like-a-human
description: Drive a real browser through bot-detection gates (Cloudflare Turnstile, hCaptcha, "verify you are human") by being a legitimate user — real fingerprint, real OS-level input events, clean residential exit. Diagnose outside-in before touching browser flags.
---

# browse-like-a-human

Make an automated browser pass anti-bot gates without defeating them. The operative idea: leave nothing to detect.

## When to use

- A signup, login, or checkout flow blocks your automation behind a "verify you are human" gate
- A page loops on an interstitial (`Just a moment...`, `Performing security verification`) and never renders content
- `cf-turnstile-response` comes back empty and the form silently refuses to submit
- You are about to write "the site blocks bots, a human has to do this" — **check the three layers first**

Do **not** use this to build CAPTCHA solvers, mass-register accounts, or evade controls on systems you have no right to access.

## Before anything else: diagnose outside-in

Read [`references/diagnosis.md`](references/diagnosis.md). The short version:

```
Layer 1  IP reputation      →  no widget rendered? → stop, fix the exit
Layer 2  Behaviour          →  widget renders, clicking does nothing? → fix input injection
Layer 3  Fingerprint        →  token stays empty? → fix the three leaks
```

Two probes decide which layer you are in:

```js
document.querySelectorAll('iframe').length
// 0 → there is no CAPTCHA here to solve. You are being rejected on reputation.

document.querySelector('[name="cf-turnstile-response"]')?.value.length
// > 0  → you have a token, the problem is downstream
// 0    → with your own browser: fingerprint/exit. In a cloud sandbox: page needs no token.
```

**Never** start by editing launch flags. Flag-tweaking is a layer-3 fix and most blocks are layer 1.

## Procedure

### 1. Capture the failure precisely

Write down, before changing anything:

- page title and URL when it stalls
- whether a challenge `iframe` exists
- the token length at submit time, if a widget is present
- how many times the Ray ID changed (a changing Ray ID = repeated fresh evaluation)
- whether the same configuration ever passed before

If a configuration passed once, **that configuration is your baseline**. Re-run it unchanged before trying anything new. Changing three variables and re-testing teaches you nothing.

### 2. Fix the exit (layer 1)

Use a residential ASN on an interface dedicated to this task. Do not reconfigure the tunnel your other services depend on — bring up a separate `tunN`, its own routing table, its own proxy port.

Verify what the site will actually see:

```bash
curl -s --interface tunN https://api.ipify.org
curl -s "https://ip.net.coffee/ip/$(curl -s --interface tunN https://api.ipify.org)"
# want: hosting=false, proxy=false, residential ISP
```

### 3. Fix the fingerprint (layer 3)

Three signals decide it. See [`references/fingerprint-hardening.md`](references/fingerprint-hardening.md).

| Signal | Must not be |
|---|---|
| `navigator.userAgentData.brands` | `null` |
| WebGL `UNMASKED_RENDERER_WEBGL` | `ERR` |
| `navigator.webdriver` | `true` |

Both sides must agree: the JS-visible values **and** the HTTP client hints (`sec-ch-ua`). Patching one and not the other is itself a detectable inconsistency.

In headless/Xvfb environments, never pass `--disable-gpu` — it makes WebGL report `ERR`. Use `--use-gl=angle --use-angle=swiftshader --enable-unsafe-swiftshader`.

### 4. Fix the input (layer 2)

Clicking and typing must arrive as real events. See [`references/human-mouse.md`](references/human-mouse.md).

- `XSendEvent` (most desktop-automation tools) → **synthetic, ignored**
- `XTEST` (`xdotool`, XInput2) → **device-level, accepted**

Add kinematics: eased path over 25–28 steps, ±1.5 px jitter, 8–22 ms between steps, a short random walk before the press, and typing with per-character delays. Then leave the page be — real users pause.

Do not mix mechanisms within one session.

### 5. Get a reusable credential, then leave the browser

Once authenticated, create an API token / session credential. Everything afterwards — creating resources, reading configuration, fetching secrets — goes through the API. A browser session is a one-time cost to obtain; it is a terrible way to do ongoing work.

If a dashboard masks a secret behind a reveal icon, do not fight the UI. Find the API:

```bash
GET /api/.../vars                  # list keys and ids (values masked)
GET /api/.../vars/<id>/reveal      # plaintext value
```

### 6. Verify with a real client before declaring success

See [`references/verification.md`](references/verification.md). A hand-built `curl` probe that returns `400` proves nothing — `400` often means the service is healthy and your request shape is wrong. Build the real configuration, exercise it, and read back an externally visible fact (exit IP, an HTTP 204, a fetched payload).

## Guardrails

Hard rules, learned the expensive way. Full write-up: [`references/operations.md`](references/operations.md).

**Diagnosis discipline**

- Re-run the last-known-good configuration before changing anything
- Change one variable per attempt
- If the identical error appears twice, stop testing and go read the mechanism
- "It clicked but nothing happened" is data — usually about *how* you clicked

**Input and UI**

- Never click icon buttons by matching empty text (`innerText === ''`). `Stop`, `Restart`, collapse arrows and reveal-eye icons are indistinguishable in DOM text. Match `aria-label`, or screenshot first, or use the API.
- Never assume a click landed. Read state back.
- A modal overlay, even an informational toast, will swallow every click on the page. Remove or dismiss it before clicking.

**Process management**

- `pgrep -f` / `pkill -f` match your own shell's command line and will kill your own command (`exit -15` / SIGTERM, truncated output). Use `pkill -x <name>` or `kill <pid>`.
- Do not wrap long-lived processes in `timeout`; the supervisor dies and takes the browser with it.
- Browser login state lives in the profile directory. Restarting with the same profile and the same fingerprint reuses the session — but clears nothing else, so a change of exit IP or UA invalidates clearance cookies.

**Reporting**

- Distinguish "my operation was wrong" from "the platform does not allow this". They look identical from the outside and only one of them justifies giving up.

## Reference index

| File | Contents |
|---|---|
| [`references/diagnosis.md`](references/diagnosis.md) | Three-layer model, probes, decision tree, what each failure means |
| [`references/fingerprint-hardening.md`](references/fingerprint-hardening.md) | Three leaks, injection, JS/HTTP consistency, WebGL setup |
| [`references/human-mouse.md`](references/human-mouse.md) | XTEST vs XSendEvent, kinematics, runnable code |
| [`references/turnstile-mechanics.md`](references/turnstile-mechanics.md) | invisible / managed / interactive, token semantics, judging pass/fail |
| [`references/operations.md`](references/operations.md) | Virtual desktop, window manager requirement, UI-click discipline, process pitfalls |
| [`references/verification.md`](references/verification.md) | Real-client validation, link round-trip, honest status reporting |
