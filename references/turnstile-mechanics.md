# Turnstile mechanics

Understanding what the widget is doing prevents most wasted retries.

## Three modes

Cloudflare Turnstile renders in one of three modes. The same site can use different modes on different pages, or escalate between them under suspicion.

| Mode | What the user sees | What happens |
|---|---|---|
| **Invisible** | nothing | widget runs, usually returns a token in 1–3 s with no interaction |
| **Managed** | nothing at first, then a checkbox appears if the score is uncertain | risk score decides whether to challenge |
| **Interactive** | a visible "Verify you are human" checkbox | explicit user interaction required |

Escalation is common: a fresh page load is invisible, then after a few suspicious signals the next challenge arrives as an interactive checkbox. **A page that shows no widget is not necessarily unprotected — the invisible widget may already have issued a token.**

### How to tell which one you got

```js
// Is there a challenge frame at all?
const frames = [...document.querySelectorAll('iframe')]
  .map(f => f.src)
  .filter(s => s.includes('challenges.cloudflare.com'));

// Is there a rendered widget container?
document.querySelectorAll('[data-sitekey], .cf-turnstile').length

// Did a token already appear without you doing anything?
document.querySelector('[name="cf-turnstile-response"]')?.value.length
```

A token with length > 0 and no visible widget = invisible mode passed on its own. That is the desired outcome and it means the fingerprint and exit are good enough that no challenge was warranted.

## The token is the only real signal

```js
document.querySelector('[name="cf-turnstile-response"]')?.value.length
```

| Value | Meaning |
|---|---|
| `0` | no token — the submission will be refused or silently dropped |
| ~500–900 chars | valid token, single use, ~300 s lifetime, **bound to the issuing IP** |

What is *not* a success signal:

- the checkbox showing a tick (the widget can be visually satisfied while producing no token)
- the page appearing normal afterwards
- a button changing to a loading state ("Sending…")

A form that says *"We couldn't confirm you're a person. Try again."* after looking normal is the canonical empty-token failure.

## Token binding

Tokens are issued against the request context:

- **IP** — a token obtained in one network context and submitted from another is rejected
- **time** — roughly five minutes
- **single use** — submitting it twice fails the second time

Practical consequence: you cannot pre-fetch tokens for later use, and you cannot move a token between sessions. The browser that holds the page must submit from the same exit.

## Site-level vs application-level verification

Two different checks run at signup flows. Confusing them sends you down the wrong path.

**Site-level (Turnstile on page load):** the whole page is protected. If this fails you never see the site's own content.

**Application-level (server verifies on submit):** the page renders fine, and the API endpoint checks the token server-side. Test directly:

```bash
curl -s -o /dev/null -w '%{http_code} %{redirect_url}\n' \
  -X POST 'https://site/api/auth/signin' \
  -H 'Content-Type: application/json' \
  -d '{"email":"you@example.com"}'
# 302 → ?error=Captcha   = server-side verification is real
# 200                   = the API does not need a token; use the API
```

Try three payloads — no token, `XXXX.DUMMY.TOKEN.XXXX`, and a random string. Identical rejection for all three means genuine `siteverify` validation. This is worth five minutes because it tells you whether a fabricated token can ever work (it cannot) and whether the API path is open (sometimes it is).

## The case where a widget renders but never issues a token

Documented because it wastes a lot of time when misread: a browser session can render the widget successfully — `turnstile.render()` returns a widget id — and still receive no token, indefinitely.

Observed in browsers hosted inside the same infrastructure as the verifier. The logic is coherent: the operator will not issue a clearance token to an IP it classifies as its own infrastructure. It does not challenge you, and it does not pass you.

**Diagnosis:**

```js
window.turnstile?.render ? 'API available' : 'not loaded'
// render succeeds, widgetId obtained, token never arrives after 60 s+
```

**Correct response:** stop. This environment will not produce a token. Switch to a real browser with a residential exit, or use the site's API if it exists. Do not keep retrying with different widget options.

## Judging outcomes without guessing

```
token length > 0            → passed; any remaining failure is downstream
token length == 0, own browser, widget rendered
                            → fingerprint (three leaks) or exit reputation
token length == 0, widget never rendered
                            → IP-level rejection; no interaction possible
token length == 0, cloud sandbox
                            → expected; the platform withheld the token
form refuses despite token  → CSRF, required hidden field, or IP changed
                              between issue and submit
```

## Practical notes

- **Do not retry after an explicit personhood rejection.** `We couldn't confirm you're a person` is deterministic for a given configuration. Change a variable or stop.
- **Long swaths of retries raise your score.** Each attempt is a data point against you. Three attempts with distinct configurations teach more than thirty identical ones.
- **The widget has its own network traffic.** Blocking `challenges.cloudflare.com` at the proxy level will prevent the widget from working at all — sometimes the "failure" is your own DNS or proxy rule.
- **Time matters.** A token issued before a system clock skew or a long pause may expire before submission. Keep issue and submit within the same interaction.

## What good looks like

With the three fingerprint leaks fixed and a residential exit:

```
page load → no visible widget → token appears within ~2 s
submit    → form accepts → next step renders
second gate (if any) → same pattern
```

No interaction, no checkbox, no retries. If you find yourself clicking a checkbox repeatedly, you are past the point where the widget can help you — the score already decided.
