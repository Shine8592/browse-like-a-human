# browse-like-a-human

**An agent skill for browser automation that passes bot detection by *being* a legitimate user — not by patching around the check.**

[中文说明](README.zh-CN.md) · [Skill definition](SKILL.md) · [References](references/)

---

## The distinction that matters

There are two ways to get a browser past Cloudflare Turnstile, hCaptcha, or a generic "verify you are human" gate.

**Bypassing** means defeating the check: harvesting a token from a solver farm, replaying a clearance cookie, patching a detection script, paying a CAPTCHA service. It works until the vendor ships a detection update — usually within weeks — and from the platform's point of view it *is* an attack. Every bypass leaves evidence that a bypass happened.

**Browsing like a human** means there is nothing left to detect. Real browser engine. Complete fingerprint. Genuine input events injected at the OS level. A clean residential exit. The platform's risk engine looks at the session and finds a normal user, because every signal it inspects is *true* — not spoofed into looking true.

```
Bypass:   fake the signals so the detector believes you're human
          → detector improves → you're broken
          → you are an anomaly in their logs

Human:    be a human as far as every observable is concerned
          → detector improves → nothing changes
          → you are indistinguishable from the median user
```

This repository is about the second path. It is the harder one to build and by far the easier one to maintain.

---

## Where automated browsers actually fail

Every "my Playwright script gets blocked" report resolves to the same three-layer stack. Failures are diagnosed outside-in — and almost everyone debugs the wrong layer.

| Layer | What the platform checks | Typical failure | Fix difficulty |
|---|---|---|---|
| **1. IP reputation** | ASN, datacenter vs residential, proxy/VPN flags, abuse history, geo consistency | AWS/Oracle/DigitalOcean IP that has been seen ten thousand times | Easy — switch exit |
| **2. Behaviour** | Mouse entropy, keystroke timing, scroll physics, focus/blur sequence, request cadence | `element.click()` teleports the cursor; `fill()` sets 500 chars in 1 ms | Medium — inject real events |
| **3. Fingerprint** | `navigator.*`, WebGL, canvas, fonts, audio, TLS/JA3, client hints consistency | `brands: null`, `webdriver: true`, WebGL renderer `ERR` | Medium — fix the leaks |

**The rule that saves the most time:** if a page never renders a Turnstile widget at all, or keeps looping on the interstitial with a new Ray ID each time, stop editing browser flags. Layer 1 failed, and layers 2 and 3 never even got a chance to be evaluated.

```js
// Diagnostic: is there anything to click, or are you being rejected before that?
document.querySelectorAll('iframe').length   // 0 → pure JS challenge, IP-level rejection
document.querySelector('[name="cf-turnstile-response"]')?.value.length
// 0 with your own browser   → fingerprint or exit is wrong (fixable)
// 0 inside a cloud sandbox  → the page simply didn't require a token
```

---

## The three fingerprint leaks that decide everything

Tested against a live two-stage Turnstile flow. With all three green plus a residential exit, **both stages passed unattended** — the first one invisibly, without a checkbox ever appearing.

| Signal | Broken value | Correct value |
|---|---|---|
| `navigator.userAgentData.brands` | `null` | `[{Google Chrome, 131}, {Chromium, 131}, {Not_A Brand, 24}]` |
| WebGL `UNMASKED_RENDERER_WEBGL` | `ERR` | `ANGLE (Intel, Intel(R) UHD Graphics 630 Direct3D11 ...)` |
| `navigator.webdriver` | `true` | `undefined` |

`brands: null` is the single most fatal of the three. No shipping Chrome ever returns null there. It is a direct confession.

And the flag that causes the second one, in headless/Xvfb setups, is `--disable-gpu`. It looks sensible and it destroys your WebGL renderer. Use software rasterisation instead:

```bash
# wrong
--disable-gpu
# right
--use-gl=angle --use-angle=swiftshader --enable-unsafe-swiftshader
```

Details, injection snippets, and consistency checks: [`references/fingerprint-hardening.md`](references/fingerprint-hardening.md)

---

## The mouse problem: XSendEvent vs XTEST

This is the single highest-value lesson in the repository.

Two ways to move a cursor on X11:

| Mechanism | What it actually does | What the risk engine sees |
|---|---|---|
| `XSendEvent` (most "desktop automation" tools) | synthesises an event and posts it to a window | a synthetic event — the window was never actually clicked |
| `XTEST` (`xdotool`, XInput2) | injects at the device/driver level, before the window system sees it | a real pointer moving and a real button press |

With `XSendEvent`, a Turnstile checkbox can be hit with pixel precision and **does not move**. Not "fails" — does not respond at all, which reads as a bot-adjacent anomaly rather than a wrong answer.

With `XTEST`, the same coordinate triggers `Verifying you are human…` and then passes.

```bash
# cua-driver and similar tools → XSendEvent → ignored
# xdotool → XTEST → accepted
xdotool mousemove --sync 812 447
sleep 0.1
xdotool click 1
```

Add human kinematics on top — bezier-ish easing, 25–28 steps, ±1.5 px jitter, 8–22 ms between steps, a small random walk before the press. Full parameters and code: [`references/human-mouse.md`](references/human-mouse.md)

> **Never mix the two mechanisms in one flow.** A single synthetic event among twenty real ones is enough to fail the session, and it produces a bug that looks like a site problem and is actually your own inconsistency.

---

## What's inside

```
SKILL.md                        the skill itself — triggers, procedure, guardrails
references/
  diagnosis.md                  the three-layer model, probes, decision tree
  fingerprint-hardening.md      the three leaks, injection, consistency checks
  human-mouse.md                XTEST vs XSendEvent, kinematics, working code
  turnstile-mechanics.md        invisible / managed / interactive, token judging
  operations.md                 desktop setup, UI-click discipline, process pitfalls
  verification.md               real-client end-to-end validation
```

---

## Quick start

```bash
# 1. virtual desktop
Xvfb :99 -screen 0 1440x900x24 -nolisten tcp &
DISPLAY=:99 openbox &            # a missing WM silently breaks all clicking

# 2. residential exit on its own interface — never touch your production tunnel
#    (bring up a dedicated tunN + routing table + proxy port)

# 3. real-browser session with the fingerprint fixed and the proxy attached
#    → see references/fingerprint-hardening.md
```

Then verify with a real client instead of guessing — see [`references/verification.md`](references/verification.md). `curl` hand-crafting a WebSocket upgrade returns `400` even when everything is healthy; `400` means the server is alive, `502/503` means it is not.

---

## Honest boundaries

This skill is for **legitimate automation as yourself**:

- creating and managing accounts you are entitled to have
- testing your own systems' anti-bot defences
- research into how detection actually works

It is explicitly **not** for:

- CAPTCHA-solving farms, token resale, or any third-party token service
- mass or throwaway account creation
- evading rate limits or access controls on systems you don't own
- anything that breaks the target's Terms of Service

The techniques here make your browser indistinguishable from a normal user. That is a property, not a loophole — and like any capability, its legitimacy comes from what you point it at. If a platform's terms forbid automated signup, being technically capable does not make it acceptable.

No credentials, tokens, personal data, or site-specific bypasses are included. Everything is described generically so it survives the next detection update.

---

## License

MIT — see [LICENSE](LICENSE).

Built from field notes: real sessions against live Cloudflare-protected signup flows, where every failed attempt was written down with its root cause. The failures are the value; the successes were mostly a matter of not making avoidable mistakes.
