# 02 — Find your own device's broker, login and topics

Everything the takeover needs — the broker IP, the MQTT login, your device ID, the topic names and the
command format — can be read **passively** from your own opener's traffic. You never send the door a
command in this phase; you only watch what it already sends.

> Do this only for a device **you own**, on **your own network**.

## What you're trying to learn

1. The **broker IP and port** the device connects out to.
2. The MQTT **username / password** and **client ID** in its CONNECT packet.
3. The **topic names** (`/hentronix/dev/<id>` and `/hentronix/app/<id>`) and the **JSON** on each.
4. Later, the **command format** for open/close and settings (a separate step — see the end).

## Where to capture

You need to see the device's traffic as it leaves your network. Easiest options:

- **On the router/gateway**, if it runs Linux and has `tcpdump` (UniFi, OPNsense/pfSense, OpenWrt,
  etc.). Capture on the interface/VLAN the opener lives on. This is the cleanest spot.
- **A machine on the same VLAN as the opener**, running `tcpdump`/Wireshark (a Pi, a NUC, a laptop). If
  the opener is on an isolated IoT VLAN, put the capture box on that VLAN, or temporarily allow it.
- **Phone‑side**: run the Hentronix app through a logging HTTP(S)/TCP proxy or a capture VPN to see the
  broker endpoint from the app's side (the app hits the same broker).

First, find the opener's IP (your router's client list, or by its MAC vendor). Then capture.

> **Gotcha — hardware offload hides traffic.** Many gateways (UniFi with its Qualcomm NSS / "ECM"
> acceleration, and others) hand **established** connections to a hardware fast path that bypasses the
> CPU, so `tcpdump` on the gateway only sees the **first second or so** of each connection. The device's
> brief idle check‑ins stay visible, but a long *awake* session — exactly where open/close happen — can
> be almost entirely invisible. If your command capture comes back empty, **temporarily disable hardware
> offload / flow acceleration** for the test, or capture from a box **on the same VLAN** as the opener
> instead of on the gateway.

## Passive capture (read‑only)

Filter to just the opener's IP so the output stays readable. Replace `<DOOR_IP>` with your device's IP.

**See where it connects (DNS + new TCP connections):**
```bash
sudo tcpdump -ni <IFACE> -nn -vvv \
  "host <DOOR_IP> and (udp port 53 or tcp[tcpflags] & tcp-syn != 0)"
```
Look for the destination `IP:port` of its outbound connections (commonly `:1883` plaintext MQTT, or
`:8883` if TLS). If you see a DNS query, note the hostname; if it jumps straight to an IP with no DNS,
the broker IP is hardcoded.

**See the MQTT payloads (login + topics + JSON):**
```bash
sudo tcpdump -ni <IFACE> -nn -A -s0 \
  "host <DOOR_IP> and tcp port 1883"
```
In the readable text you'll spot:
- the `MQTT` CONNECT with the **client ID**, **username** and **password** (plaintext, if port 1883);
- a **SUBSCRIBE** to `/hentronix/app/<id>` (its command topic);
- a **PUBLISH** to `/hentronix/dev/<id>` with a JSON status blob;
- a **retained** message coming back on `/hentronix/app/<id>`.

Write those down — they're your `<BROKER_IP>`, `<MQTT_USER>`, `<MQTT_PASS>`, `<DEVICE_ID>`.

> If it's on `:8883` (TLS) instead, you can't read the payload this way. Check whether it validates the
> certificate (pins). If it doesn't pin, you can still redirect and present your own cert; if it pins,
> the clean route is reflashing the ESP to open firmware (see [03](03-home-assistant-takeover.md),
> "Fallbacks").

## Record the JSON field meanings (still passive)

Watch the telemetry topic for a while and note which fields change:
- a field that **increments ~once per second** is an uptime counter;
- a field that tracks **door open vs closed** is your state flag;
- a **voltage‑like** number that matches the "live voltage" shown in the app is the battery. Read the
  app's "live voltage" / "previous‑night voltage" at the same time to match them up.
- a **door‑state** field often has more than two values — distinct codes for *closed*, *closing*,
  *opening* and *open* — so you can show motion, not just the endpoints. Match them by watching the field
  while the door actually moves. A temperature reading (also ×10, matching the app) is often in there too.

## Learn the command format — manually, before you switch anything over

The scheduled dawn/dusk moves are triggered by the device's own clock, so they **don't** send a command
over `/app/` — watching them only teaches you the *state* fields. To capture **commands**, run the
payload capture while **you operate your own door from the real app**: press open, then close, then
change each setting once, pausing between presses. Each press appears on `/hentronix/app/<id>`; diff it
against the idle retained message to see which field(s) carry the command.

What you'll typically find (confirm the exact codes on **your** unit):

- a **command field** plus a **value field** carry the action — often one "move" command with the value
  choosing **open** vs **close** — with separate codes for **wake**, **heartbeat/keep‑alive**, **sleep**
  and extras like a light/torch.
- the device reports progress back on `/dev/` via its door‑state field (closing → closed, etc.).

> **Gotcha — a counter/nonce field may be required.** On the unit tested, every command the app sent
> carried a small, changing integer field (it looked like a counter or signal value). **Commands that
> omitted it were silently ignored and the door stayed asleep.** Its exact value wasn't checked (any
> changing number worked), but the field had to be **present**. If replayed commands do nothing,
> reproduce *every* field the app sends — including ones that look incidental.

Do this **while the vendor cloud is still up** (the app needs it) and **before** you add the redirect in
[03](03-home-assistant-takeover.md) — once traffic is redirected, the app can no longer reach the door.

You now have everything to build the local takeover.
