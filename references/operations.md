# Operations: desktop, clicks, processes

Everything here was learned by breaking something. Each rule exists because ignoring it cost time.

## Virtual desktop

A headless X server plus a window manager is the minimum for real input delivery.

```bash
Xvfb :99 -screen 0 1440x900x24 -nolisten tcp &
export DISPLAY=:99
openbox &                     # NOT optional
```

**A missing window manager silently breaks clicking.** Without a WM, focus and stacking are unmanaged; clicks are delivered to the wrong place or dropped entirely, with no error anywhere. If "the click landed precisely but nothing happened" and you have already ruled out `XSendEvent`, check that a WM is running.

Verify the desktop is usable before starting a browser:

```bash
xdotool getdisplaygeometry          # expect 1440 900
xdpyinfo | grep dimensions
```

## Clicking discipline

### Never click icon buttons by empty text

The failure that cost an afternoon:

```js
// DO NOT DO THIS
[...document.querySelectorAll('button')]
  .filter(b => b.innerText.trim() === '')      // ← matches Stop, Restart, collapse, reveal-eye
  .forEach(b => b.click());                    // ← clicked 15 buttons
```

Result: the cloud application was stopped mid-session. An `innerText`-based matcher cannot distinguish a "reveal secret" eye icon from a "Stop service" button — both are text-free.

**Rules:**

1. Match `aria-label`, `title`, or a stable class — never empty text
2. Screenshot and look before any state-changing action
3. Click one thing, then read state back
4. **If an API exists, do not click at all** — see below

### If an API exists, use it

Dashboard secrets are usually masked (`•••••`) behind a reveal toggle. Fighting the UI costs clicks and risks casualties. The same data is almost always one request away:

```bash
# list: keys and ids visible, values masked
GET /api/.../vars

# reveal: plaintext, one id at a time
GET /api/.../vars/<id>/reveal
```

Two requests, no clicking, no risk of pressing something destructive. Before automating any dashboard, spend five minutes looking for `/api/`.

### Modal overlays swallow everything

A `dialog-overlay` — even from an informational toast — intercepts pointer events for the entire page. The symptom is a click that raises `intercepts pointer events` and does nothing.

```js
// remove informational toasts and zombie overlays first
document.querySelectorAll('[data-slot=toast]').forEach(el => el.remove());

// then confirm the target dialog is genuinely visible
const dlg = document.querySelector('[data-slot=dialog-content]');
dlg && dlg.offsetParent !== null && dlg.hasAttribute('data-open');
```

A dialog can carry `data-open` while `offsetParent === null` — on screen it is gone, in the DOM it is "open". Calling `.click()` on its button then does nothing at all, silently, which reads as "the button is broken".

**Always verify visibility before clicking, and verify state after.**

### Element must be in the viewport

A checkbox below the fold cannot be clicked, and the widget reports a generic failure rather than "out of view".

```python
page.evaluate("el => el.scrollIntoView({block: 'center'})", handle)
time.sleep(random.uniform(0.3, 0.8))
```

## Process management

### `pgrep -f` / `pkill -f` will kill your own command

```bash
# BROKEN — matches the shell running this very command
pkill -f "my_script"
# → exit_code -15 / SIGTERM, output truncated, looks like an unrelated failure
```

`-f` matches the full command line, and your own `bash -c '...'` command line contains the pattern.

```bash
pkill -9 -x chrome                 # by exact process name
kill <pid>                         # if you have the pid, just kill it
pgrep -c -x chrome                 # counting too
pgrep -a -f "pattern"              # only when fuzzy matching is unavoidable:
                                   # look at the hit list before acting
```

Symptom to recognise: a command that ends with `exit_code: -15`, `SIGTERM`, or visibly truncated output, seemingly at random. That is self-termination, not a flaky task.

### Do not wrap long-lived processes in `timeout`

```bash
timeout 300 python browser_session.py      # dies at 5 minutes
```

When the supervisor is killed, the child browser dies with it. Mid-signup that is a lost session.

Use a proper background job without a timeout. And note the mitigating fact:

**Login state lives in the browser profile directory.** Restarting with the same profile and the same fingerprint parameters restores the session — often straight back into the logged-in dashboard, no re-authentication. But a change of exit IP or user agent invalidates clearance cookies, so "same profile" must mean same UA, same platform, same exit.

### Orphan cleanup

Before starting a new browser session, confirm no old instance is holding the port or the profile:

```bash
ss -lntp | grep 9222 || echo "port free"
pgrep -c -x chrome
```

Two browsers sharing a profile directory corrupt it.

## Slow exits

Residential tunnels are slow, and default timeouts are not patient enough. A navigation that would take 200 ms on a datacenter line can take 15 s through a residential hop.

```python
page.goto(url, wait_until="commit", timeout=120_000)
```

`wait_until="commit"` returns as soon as the navigation commits rather than waiting for load events, which may never fire while resources are still streaming. The usual default (`load` / `domcontentloaded`) times out and raises, and the traceback reads like a script bug rather than a network condition.

Symptom to recognise: browser exits immediately after launch, logs show a navigation timeout on an innocuous URL. It is not a crash.

Measure the tunnel before blaming code:

```bash
curl -s -o /dev/null -w '%{time_total}s\n' --interface tunN https://www.google.com
```

## Cleanup checklist

```
[ ] browser processes terminated        pgrep -c -x chrome
[ ] debug port released                 ss -lntp | grep 9222
[ ] virtual desktop still running       (keep it if you will reuse it)
[ ] dedicated tunnel kept or torn down deliberately — never leave
    a half-configured interface attached to production routes
[ ] credentials moved out of shell history into a 600 file
```

## Reporting

Distinguish, in your own notes and in anything you tell a user:

- **my operation was wrong** — coordinate space, synthetic events, blind clicking, killed my own process
- **the platform refuses** — IP reputation, application-level verification, withheld token

From the outside these look identical: "it didn't work". Only the first is your bug, and only the second justifies escalation to a human.
