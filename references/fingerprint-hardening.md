# Fingerprint hardening

Three signals decide most outcomes. A fourth consistency rule (JS vs HTTP headers) decides the rest.

## The three leaks

### 1. `navigator.userAgentData.brands` — the fatal one

```js
navigator.userAgentData  // Playwright Chromium out of the box
// undefined, or { brands: [], mobile: false, platform: '' } → null-ish
```

A shipping Chrome **always** returns a populated brands array:

```js
[
  { brand: 'Google Chrome', version: '131' },
  { brand: 'Chromium', version: '131' },
  { brand: 'Not_A Brand', version: '24' }
]
```

Returning `null`, `[]`, or leaving the property undefined is the loudest possible self-report. No real user's browser does this. Fix it before anything else.

### 2. WebGL renderer

```js
const c = document.createElement('canvas');
const gl = c.getContext('webgl');
const dbg = gl.getExtension('WEBGL_debug_renderer_info');
gl.getParameter(dbg.UNMASKED_RENDERER_WEBGL);
```

| Value | Verdict |
|---|---|
| `ERR` / `null` / throws | Broken — and impossible for a real machine |
| `ANGLE (Intel, Intel(R) UHD Graphics 630 Direct3D11 vs_5_0 ps_5_0, D3D11)` | Fine |
| `Google SwiftShader` | Acceptable fallback (real VMs and older machines report it) |

**Root cause in headless environments is `--disable-gpu`.** It is nearly universal in container/Xvfb setups and it eliminates WebGL entirely. Replace it:

```bash
# never
--disable-gpu

# instead
--use-gl=angle --use-angle=swiftshader --enable-unsafe-swiftshader
```

`--use-gl=angle --use-angle=swiftshader` gives you a software rasteriser that still reports a plausible renderer string. `--enable-unsafe-swiftshader` silences the deprecation gate that would otherwise disable it again on newer Chromium builds.

### 3. `navigator.webdriver`

```js
navigator.webdriver   // Playwright / Selenium: true
```

Must be `undefined`. `false` is also wrong — a real browser does not define it at all. An explicitly-defined `false` is a detectable tell that something assigned it.

## The consistency rule

Anything you fix in JavaScript must also be fixed in the HTTP request headers. Detectors compare the two, and a mismatch is worse than a uniform default because it indicates deliberate tampering.

| JS surface | HTTP surface |
|---|---|
| `navigator.userAgent` | `User-Agent` |
| `navigator.userAgentData.brands` | `sec-ch-ua` |
| `navigator.userAgentData.platform` | `sec-ch-ua-platform` |
| `navigator.userAgentData.mobile` | `sec-ch-ua-mobile` |

A Linux UA reporting `platform: "Win32"` in JS while sending `sec-ch-ua-platform: "Linux"` is a contradiction no real browser can produce.

## Injection — both surfaces

### JavaScript side (Playwright)

```python
STEALTH_JS = r"""
// webdriver: delete entirely, do not set to false
delete Object.getPrototypeOf(navigator).webdriver;
try { Object.defineProperty(Navigator.prototype, 'webdriver', { get: () => undefined }); } catch (e) {}

// userAgentData with plausible brands
const BRANDS = [
  { brand: 'Google Chrome', version: '131' },
  { brand: 'Chromium', version: '131' },
  { brand: 'Not_A Brand', version: '24' },
];
const realUAD = navigator.userAgentData;
const fakeUAD = {
  brands: BRANDS,
  mobile: false,
  platform: 'Windows',
  getHighEntropyValues: async (hints) => ({
    brands: BRANDS, mobile: false, platform: 'Windows',
    platformVersion: '15.0.0', architecture: 'x86', bitness: '64',
    model: '', uaFullVersion: '131.0.0.0', fullVersionList: BRANDS,
  }),
  toJSON() { return { brands: BRANDS, mobile: false, platform: 'Windows' }; },
};
Object.defineProperty(Navigator.prototype, 'userAgentData', { get: () => fakeUAD });

// platform must match the UA string
Object.defineProperty(Navigator.prototype, 'platform', { get: () => 'Win32' });

// plugins: a headless browser has none; a desktop Chrome has these
const plug = (name, filename, desc) => ({ name, filename, description: desc, length: 1 });
Object.defineProperty(Navigator.prototype, 'plugins', {
  get: () => [
    plug('PDF Viewer', 'internal-pdf-viewer', 'Portable Document Format'),
    plug('Chrome PDF Viewer', 'internal-pdf-viewer', 'Portable Document Format'),
    plug('Chromium PDF Viewer', 'internal-pdf-viewer', 'Portable Document Format'),
    plug('Microsoft Edge PDF Viewer', 'internal-pdf-viewer', 'Portable Document Format'),
    plug('WebKit built-in PDF', 'internal-pdf-viewer', 'Portable Document Format'),
  ],
});

// languages
Object.defineProperty(Navigator.prototype, 'languages', { get: () => ['en-US', 'en'] });

// permissions query is a classic tell
const origQuery = window.navigator.permissions?.query?.bind(navigator.permissions);
if (origQuery) {
  navigator.permissions.query = (p) =>
    p && p.name === 'notifications'
      ? Promise.resolve({ state: Notification.permission })
      : origQuery(p);
}
"""

context.add_init_script(STEALTH_JS)   # runs before any page script
```

`add_init_script` matters: it executes before page JavaScript, so detection code cannot snapshot the pristine values first.

### HTTP side (CDP)

```python
context = browser.new_context(...)
page = context.new_page()
cdp = context.new_cdp_session(page)
cdp.send("Emulation.setUserAgentOverride", {
    "userAgent": ("Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 "
                  "(KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36"),
    "acceptLanguage": "en-US,en;q=0.9",
    "platform": "Win32",
    "userAgentMetadata": {
        "brands": [
            {"brand": "Google Chrome", "version": "131"},
            {"brand": "Chromium", "version": "131"},
            {"brand": "Not_A Brand", "version": "24"},
        ],
        "fullVersionList": [
            {"brand": "Google Chrome", "version": "131.0.0.0"},
            {"brand": "Chromium", "version": "131.0.0.0"},
            {"brand": "Not_A Brand", "version": "24.0.0.0"},
        ],
        "fullVersion": "131.0.0.0",
        "platform": "Windows",
        "platformVersion": "15.0.0",
        "architecture": "x86",
        "model": "",
        "mobile": False,
        "bitness": "64",
        "wow64": False,
    },
})
```

`setUserAgentOverride` with `userAgentMetadata` is what makes `sec-ch-ua*` headers agree with the JS objects. Without it you get JavaScript claiming Chrome 131 while the network layer announces HeadlessChrome.

## Verification harness

Run this after setup, before touching the target site. All four must pass.

```python
checks = page.evaluate("""() => {
  const gl = document.createElement('canvas').getContext('webgl');
  const dbg = gl && gl.getExtension('WEBGL_debug_renderer_info');
  const renderer = dbg ? gl.getParameter(dbg.UNMASKED_RENDERER_WEBGL) : 'NO_WEBGL';
  const uad = navigator.userAgentData;
  return {
    ua: navigator.userAgent,
    webdriver: navigator.webdriver,
    brands: uad ? uad.brands : null,
    platform: uad ? uad.platform : null,
    navPlatform: navigator.platform,
    renderer,
    plugins: navigator.plugins.length,
    langs: navigator.languages,
  };
}""")

assert checks['webdriver'] is None
assert checks['brands'], "brands is empty — fatal"
assert checks['renderer'] not in ('ERR', 'NO_WEBGL', None)
assert 'HeadlessChrome' not in checks['ua']
assert checks['navPlatform'] == 'Win32'
assert checks['plugins'] >= 3
```

Then check the header side from inside the page:

```js
// fetch a reflection endpoint and read back what the server received
await fetch('https://httpbin.org/headers').then(r => r.json())
// verify sec-ch-ua, sec-ch-ua-platform, user-agent all agree
```

## Secondary signals worth knowing

Not usually decisive on their own, occasionally checked in combination:

- **`Notification.permission`** — headless defaults differ; `permissions.query` inconsistency is a known tell
- **Canvas hash** — SwiftShader produces a stable but unusual hash; acceptable, since VMs do too
- **Audio context fingerprint** — same reasoning
- **Missing `chrome` runtime object** — `window.chrome && window.chrome.runtime` should exist in Chromium
- **Timezone vs geo** — `Intl.DateTimeFormat().resolvedOptions().timeZone` must match the exit's country
- **WebRTC local IP leak** — can expose a datacenter or private address that contradicts the proxy

Practical priority: fix the three leaks, then the JS/header consistency, then the timezone. Stop there unless something specific keeps failing.

## What not to bother with

Once the three leaks are green and the exit is residential, a real two-stage Turnstile flow passes **unattended** — the first stage invisibly, with no checkbox ever appearing. If you still fail at that point, the problem is almost certainly layer 1 (exit), not another fingerprint field. Adding more spoofing on top of a rejected IP does not help and increases your tampering score.
