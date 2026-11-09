# Outside-in diagnosis

Bot detection is a stack, not a single check. Almost every wasted afternoon comes from debugging the wrong layer.

## The stack

```
┌─────────────────────────────────────────────────────┐
│  L1  IP reputation                                  │  evaluated first
│      ASN class, datacenter vs residential,          │  cheapest for the
│      proxy/VPN flags, abuse history, geo match      │  platform to check
├─────────────────────────────────────────────────────┤
│  L2  Behaviour                                      │  evaluated only if
│      mouse entropy, keystroke timing, scroll        │  L1 passes
│      physics, focus/blur, request cadence           │
├─────────────────────────────────────────────────────┤
│  L3  Fingerprint                                    │  evaluated only if
│      navigator.*, WebGL, canvas, audio, fonts,      │  L1 (and L2) pass
│      TLS/JA3, JS-vs-HTTP-header consistency         │
└─────────────────────────────────────────────────────┘
```

The platform evaluates in this order because each layer is progressively more expensive: an ASN lookup is a dictionary hit, behavioural scoring needs a model, fingerprinting needs to render the page.

**Consequence:** if L1 fails, L2 and L3 are never consulted. Your meticulously patched `userAgentData` is not being inspected at all.

## Probe 1 — is there anything to interact with?

```js
document.querySelectorAll('iframe').length
```

| Result | Meaning | Go to |
|---|---|---|
| `0`, and the page is stuck on an interstitial | **No CAPTCHA is present.** You are being rejected on reputation. There is no checkbox to click. | L1 |
| `> 0` with a challenge host (`challenges.cloudflare.com`, `hcaptcha.com`, …) | A real widget is rendered and can be interacted with | L2 / L3 |
| `0`, and the site's own content is rendered | No gate on this request. Your problem is elsewhere — session, cookies, CSRF, an app-layer check | none of these |

A changing `Ray ID` or `cf_clearance` round-trip on every reload means the platform is re-evaluating you from scratch each time — a strong signal you are failing at L1.

## Probe 2 — did you get a token?

```js
document.querySelector('[name="cf-turnstile-response"]')?.value.length
```

This is the only honest pass/fail signal on a Turnstile-protected form. Not the checkbox ticking, not the page looking normal, not the button turning into "Sending…".

| Result | In your own browser | In a cloud-hosted browser |
|---|---|---|
| `0` | Fingerprint or exit is wrong — **fixable** | The page did not require a token; this `0` is not a failure |
| `700+` (Turnstile tokens are long) | You passed | You passed |

Same number, opposite meaning. Establish which browser you are in before drawing a conclusion.

**Checkbox ticked ≠ passed.** The clearance cookie opens the page; the token is the identity proof for this specific submission. A form that displays normally and then refuses to send has an empty token.

## Probe 3 — does the widget render but refuse to score?

Diagnostic sequence when a widget is present but the token never arrives:

```js
// 1. three leaks (see fingerprint-hardening.md)
navigator.webdriver
navigator.userAgentData?.brands
// 2. WebGL renderer
const gl = document.createElement('canvas').getContext('webgl');
gl.getExtension('WEBGL_debug_renderer_info');
gl.getParameter(gl.getExtension('WEBGL_debug_renderer_info').UNMASKED_RENDERER_WEBGL);
// 3. can the widget even initialise?
typeof window.turnstile            // undefined → not loaded
window.turnstile?.render ? 'api ok' : ''
```

A widget that renders, spins, and returns an empty token while the page says *"We couldn't confirm you're a person"* is the classic L3-or-exit failure. A widget whose `turnstile.render()` succeeds but which never produces a token at all is the different, harder case documented in [`turnstile-mechanics.md`](turnstile-mechanics.md) — the platform has decided not to issue one.

## Probe 4 — is it the application, not the anti-bot layer?

Test the endpoint directly, without a browser:

```bash
curl -s -o /dev/null -w '%{http_code}\n' -X POST 'https://site/api/auth/signin' \
  -H 'Content-Type: application/json' -d '{"email":"you@example.com"}'
# 302 → /login?error=Captcha   means the server verifies the token server-side
```

If the API rejects requests that carry no token or a fabricated token, the check is genuinely server-verified. That is important: it tells you a fake `cf-turnstile-response` string will never work, and it tells you a valid token must come from a browser session whose IP matches the submission.

It also tells you something more useful — **if the API endpoints answer `200` while the pages answer `403`, you may not need a browser at all.**

## Decision tree

```
Page loads?
├─ No, 403 / connection reset
│   └─ Try the API directly (curl). Many products gate the HTML but not /api/*.
│      └─ API works? → skip the browser entirely.
│
├─ Stuck on interstitial, no iframe, Ray ID changes
│   └─ L1. Change the exit to residential. Do not touch browser flags.
│
├─ Interactive widget present, click does nothing at all
│   └─ L2. Your click is synthetic (XSendEvent). Switch to XTEST.
│
├─ Widget spins, then "couldn't confirm you're a person"
│   └─ L3. Check the three leaks, then the exit's proxy/hosting flags.
│
├─ Token exists but the form still fails
│   └─ Not a bot problem. CSRF, session binding, IP mismatch between
│      token issue and submission, or a required hidden field.
│
└─ Everything passes but no email / no result arrives
    └─ Application layer. Check spam, then check whether the send endpoint
       was actually called (network tab), before blaming detection.
```

## Anti-patterns

**Tuning flags on an L1 failure.** The most expensive mistake. Symptom: a page that never renders the widget.

**Changing several variables per attempt.** You learn nothing and lose the ability to attribute the outcome.

**Retrying an identical configuration after an explicit rejection.** Deterministic refusals are not flaky. If the same error text appears twice, the configuration is wrong, and more attempts only raise the risk score.

**Trusting a stored clearance cookie.** `cf_clearance` is bound to IP and user agent. Move the session, change the UA, and it is dead — even if the local cookie jar shows a future expiry date. The server decides, not your SQLite file.

**Blaming the platform for your own inconsistency.** "It clicked precisely and nothing happened" is almost always XSendEvent. "It worked yesterday" is almost always a changed exit or UA.
