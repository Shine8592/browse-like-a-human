# Human input on X11 — XTEST vs XSendEvent

The most expensive lesson in this repository. Two mechanisms move a cursor. One is invisible to anti-bot systems. The other is instantly recognised — and produces a failure that looks exactly like "the site is broken".

## The two mechanisms

| | `XSendEvent` | `XTEST` |
|---|---|---|
| Layer | X protocol — synthesises an event, posts it to a window | device/driver level, injected before the window system |
| Typical users | most "desktop automation" libraries, RPA tools, some AI computer-use drivers | `xdotool`, `xdotool click`, XInput2 test extension |
| What page JS sees | `event.isTrusted === false` | `event.isTrusted === true` |
| Turnstile behaviour | **no reaction at all** | reacts: spinner → pass or fail |

`isTrusted` cannot be faked from JavaScript. It is set by the browser based on whether the event came from the real input pipeline. A page can check it in one line, and anti-bot widgets do.

**The tell:** if a click lands with pixel-perfect accuracy on a checkbox and the checkbox does not move, does not animate, does not even flash — that is XSendEvent being ignored. It is not a coordinate problem and not a selector problem. The event was recognised as synthetic and discarded.

## Identify what your tool is doing

```bash
# direct XTEST: reliable
which xdotool && xdotool getmouselocation

# a driver you are unsure about — test the distinction directly
# 1. move the pointer with the tool to a known coordinate
# 2. read it back with an independent XTEST-based tool
xdotool getmouselocation
# if the driver moved it and xdotool sees the new position, the driver at least
# reached the X server; verify by clicking a page element that logs isTrusted
```

Minimal `isTrusted` probe for any HTML page:

```js
document.addEventListener('click', e => console.log('trusted:', e.isTrusted), true);
```

With a good driver every click logs `true`. With `XSendEvent` you get `false`, or no event at all if the target element swallows untrusted input.

## Driver-level input with xdotool

```bash
export DISPLAY=:99

# absolute move, waits for the server to acknowledge
xdotool mousemove --sync 812 447
sleep 0.12
xdotool click 1
```

Install:

```bash
apt-get install -y xdotool          # tiny, no daemon
```

`--sync` matters: without it, the move is queued and a following `click` can be delivered at the old position.

## Human kinematics

A real pointer does not travel in a straight line at constant speed. It accelerates, overshoots slightly, corrects, and jitters. Reproduce that:

| Parameter | Range that works |
|---|---|
| Path | cubic ease-in-out: `e = 3t² − 2t³` |
| Steps | 25–28 |
| Jitter | ±1.5 px perpendicular to the path |
| Step interval | 8–22 ms, randomised |
| Pre-click excursion | 2–3 small random segments near the target |
| Settle before press | 40–120 ms |

### Working implementation

```python
import subprocess, random, time, math

def _xdo(*args):
    subprocess.run(["xdotool", *map(str, args)], check=False)

def human_move(x_target, y_target, steps=None, jitter=1.5):
    """Eased multi-step move with perpendicular jitter."""
    out = subprocess.run(["xdotool", "getmouselocation"],
                         capture_output=True, text=True).stdout
    x0 = int(out.split("x:")[1].split()[0])
    y0 = int(out.split("y:")[1].split()[0])
    steps = steps or random.randint(25, 28)

    for i in range(1, steps + 1):
        t = i / steps
        # ease-in-out cubic
        e = 3 * t * t - 2 * t * t * t
        x = x0 + (x_target - x0) * e
        y = y0 + (y_target - y0) * e
        # perpendicular jitter, damped near the end
        damp = 1.0 - t * 0.75
        x += random.uniform(-jitter, jitter) * damp
        y += random.uniform(-jitter, jitter) * damp
        _xdo("mousemove", int(x), int(y))
        time.sleep(random.uniform(0.008, 0.022))

def human_click(x, y):
    # small random walk to the vicinity first
    for _ in range(random.randint(2, 3)):
        human_move(x + random.randint(-35, 35), y + random.randint(-25, 25),
                   steps=random.randint(8, 14), jitter=2.0)
        time.sleep(random.uniform(0.05, 0.18))
    human_move(x, y)
    time.sleep(random.uniform(0.04, 0.12))
    _xdo("click", 1)
    time.sleep(random.uniform(0.3, 0.7))
```

### Typing

`fill()` sets the value in one shot — a 40-character field completed in 0 ms is not a human. Use per-key delays, and add a small probability of a correction:

```python
def human_type(text, base=0.08, jitter=0.06):
    for ch in text:
        _xdo("type", "--delay", str(int(base * 1000)))
        time.sleep(random.uniform(base * 0.5, base + jitter))
```

For anything that matters, prefer real key events over `element.fill()`:

```python
page.click('input[type="email"]')
time.sleep(random.uniform(0.2, 0.5))
for ch in email:
    page.keyboard.type(ch, delay=random.randint(60, 160))
```

## Coordinates: window-relative vs screen-absolute

Different tools disagree, and the error is silent.

- `xdotool mousemove` → **screen absolute**
- most window-targeting automation APIs (`click {pid, x, y}`) → **window-relative**

If every click lands offset by a constant (the window origin), you are passing screen coordinates to a relative API. Read the window geometry once and subtract:

```bash
xdotool getwindowgeometry --shell $(xdotool getactivewindow)
# WINDOW=..., X=20, Y=20, WIDTH=1288, HEIGHT=851
# screen = window-relative + (X, Y)
```

Verify the cursor actually moved where you intended before clicking:

```bash
xdotool getmouselocation
```

## Requirements beyond the event mechanism

- **A window manager must be running.** On a bare Xvfb with no WM, clicks frequently never reach the client. Run `openbox` (or any WM) on the same display.
- **The element must be inside the viewport.** A checkbox scrolled below the fold cannot be clicked, and the widget will report a misleading failure. Scroll it into view first:

```python
page.evaluate("el => el.scrollIntoView({block: 'center'})", element_handle)
time.sleep(random.uniform(0.3, 0.8))   # let the scroll settle
```

- **Do not mix mechanisms.** If your session is XTEST, do not let a driver inject one synthetic event mid-flow. A single `isTrusted: false` click in a sequence of real ones is both a failure and a puzzle you will spend an hour on.

## Why this is the right approach

The alternative — finding a token, or a JS patch that makes a detector change its mind — is a race against the vendor's next release. Injecting real events is not a trick against the detector; it is the same input path a physical mouse uses. There is nothing to detect, and nothing to break when the detection model improves.

## Checklist

```
[ ] window manager running on the display
[ ] clicks delivered via XTEST (isTrusted === true)
[ ] target element scrolled into viewport
[ ] coordinates converted to the API's expected space, verified with getmouselocation
[ ] eased path, jitter, randomised intervals
[ ] per-character typing delays
[ ] only one injection mechanism in the whole session
```
