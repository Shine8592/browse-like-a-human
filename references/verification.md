# Verification before delivery

"It's set up" and "it works" are different claims. This file is about producing evidence for the second one.

## The trap: probes that look meaningful and aren't

### `curl` on a WebSocket-protected service

```bash
curl -s -o /dev/null -w '%{http_code}\n' \
  -H 'Connection: Upgrade' -H 'Upgrade: websocket' \
  -H 'Sec-WebSocket-Version: 13' -H 'Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==' \
  https://host/path
# 400
```

`400` here does **not** mean the service is broken. A healthy VLESS-over-WebSocket server rejects a hand-built upgrade request because the payload is not a valid protocol frame. Read the status class instead:

| Status | Meaning |
|---|---|
| `400` / `404` | the server is alive and responding — your request shape is wrong |
| `502` / `503` / `504` | the service is down or not routing |
| timeout / connection refused | network or TLS problem, not protocol |

So `400` proves liveness, nothing more. It cannot prove the node works.

### UI state that contradicts itself

In one session, a platform's CLI reported `status: running` in one subcommand while another reported `no_pod: No running container`. The API was right; the CLI path was wrong. **When two control surfaces disagree, believe the one that returns structured data, and cross-check with an external probe** (a plain HTTPS request to the public endpoint).

## Real client verification

Build the actual configuration and exercise it. The example below validates a VLESS + WebSocket + TLS endpoint end to end.

```python
import json, subprocess, time

cfg = {
    "log": {"loglevel": "warning"},
    "inbounds": [{
        "listen": "127.0.0.1", "port": 10899, "protocol": "socks",
        "settings": {"udp": False, "auth": "noauth"},
    }],
    "outbounds": [{
        "protocol": "vless",
        "settings": {"vnext": [{
            "address": HOST, "port": 443,
            "users": [{"id": CLIENT_ID, "encryption": "none", "level": 0}],
        }]},
        "streamSettings": {
            "network": "ws",
            "security": "tls",
            "tlsSettings": {"serverName": HOST, "allowInsecure": False},
            "wsSettings": {"path": "/" + WS_PATH, "headers": {"Host": HOST}},
        },
    }],
}
json.dump(cfg, open("/tmp/verify.json", "w"), indent=2)

proc = subprocess.Popen(["xray", "run", "-c", "/tmp/verify.json"],
                        stdout=open("/tmp/verify.log", "w"), stderr=subprocess.STDOUT)
time.sleep(4)
try:
    r = subprocess.run(
        ["curl", "-s", "--max-time", "25", "--socks5-hostname", "127.0.0.1:10899",
         "https://api.ipify.org"], capture_output=True, text=True)
    print("exit IP:", r.stdout.strip())

    r2 = subprocess.run(
        ["curl", "-s", "-o", "/dev/null", "-w",
         "connect=%{time_connect}s tls=%{time_appconnect}s total=%{time_total}s "
         "speed=%{speed_download}B/s code=%{http_code}",
         "--max-time", "40", "--socks5-hostname", "127.0.0.1:10899",
         "https://www.google.com/generate_204"], capture_output=True, text=True)
    print(r2.stdout)
finally:
    proc.terminate()
```

Read the output for facts, not impressions:

```
exit IP    → an address you can look up; should match the expected region/ASN
code 204   → the tunnel actually reaches the internet
tls=0.67s  → handshake cost, useful for spotting a bad route
speed      → throughput you can quote honestly
```

## Round-trip the shareable artifact

Verifying "the parameters I used" is not the same as verifying "the link I'm about to hand someone". Parse the delivered artifact back into a configuration and test that:

```python
import urllib.parse as up

link = "vless://<id>@<host>:443?encryption=none&security=tls&sni=...&type=ws&host=...&path=%2F..."
body, _, _frag = link[8:].partition("#")
creds, _, rest = body.partition("@")
hostport, _, qs = rest.partition("?")
host, port = hostport.rsplit(":", 1)
q = dict(up.parse_qsl(qs))

# rebuild config from q and test exactly as above
```

If the round-tripped configuration works, the artifact is correct. This catches URL-encoding mistakes (an unencoded `/` in `path` breaks some clients), a missing `sni`, or `host`/`sni` pointing at different names.

## Claim checklist before saying "done"

```
[ ] external probe answers          (HTTP response from the public endpoint)
[ ] real client reaches the internet through it
[ ] exit location matches the intent (region, ASN type)
[ ] the exact artifact to be delivered was parsed back and re-tested
[ ] quota / limits / idle policy known and stated
[ ] secret material written to a 600 file, not into chat or logs
```

## Honest reporting

Report the distinction between what you measured and what you assume:

- **measured**: exit IP, status codes, timings, throughput, page titles
- **assumed**: that the service will still be up tomorrow, that a free tier will stay free, that a tunnel will persist

Free tiers have idle policies. Tunnels drop. Say so in the delivery message rather than letting the user discover it.

If verification fails, say what failed and with which observation — not a summary judgement. "Handshake completes but the exit IP is a datacenter ASN" is actionable. "It didn't work" is not.

And if the failure is your own mistake — synthetic clicks, wrong coordinate space, a process you killed yourself — say that plainly. Users distinguish "my tooling was wrong" from "the platform forbids this" far better than they are sometimes given credit for, and only one of those is a reason to stop.
