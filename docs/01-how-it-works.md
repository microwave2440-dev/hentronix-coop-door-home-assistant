# 01 — How a Hentronix WiFi opener works

## The device

- A battery/solar **WiFi door opener** built on an **ESP‑class microcontroller** (lwIP network stack;
  replies to ping with an initial TTL of 255). It spends most of its life **asleep** to save power and
  wakes briefly to act and to phone home.
- Setup is the usual two‑phase ESP pattern: out of the box (or after the on‑board factory‑reset button)
  it becomes its **own WiFi access point**; you join it, hand it your home WiFi credentials, and it then
  joins your network as a normal client.
- The **dawn/dusk open/close schedule runs locally** on the device's own clock (RTC) and battery. This
  does **not** depend on the cloud or the app — it keeps working through an internet outage or after the
  vendor's servers disappear. The cloud only adds *remote* control, schedule editing and monitoring.

## The control model: outbound MQTT to a cloud broker

The opener does **not** listen for the app on your LAN (a port scan of the device shows no open TCP
ports). Instead it **dials out** to a **cloud MQTT broker** and the phone app connects to the **same
broker**; the broker relays between them:

```
  phone app  ─────▶  cloud MQTT broker  ◀─────  coop door
             commands │                │ commands
             status   ◀                ▶ status
```

On the unit investigated, that connection was:

- **Plain MQTT on TCP port 1883** — **no TLS / no encryption / no certificate pinning**.
- To a **hardcoded IP address** (an AWS EC2 host) — the device makes **no DNS lookup**, it just connects
  to the baked‑in IP. (So there is no hostname to redirect; the takeover works at the IP level instead.)
- Short‑lived sessions roughly every ~16 seconds: connect → subscribe to its command topic → publish a
  status message → sleep → reconnect.

> **A second, encrypted channel exists.** Alongside MQTT, the unit also makes occasional **HTTPS** calls
> to a vendor cloud API (an AWS API Gateway host). It's encrypted, so its purpose is unverified — likely
> push notifications or a "who wants me?" poll. In testing, local MQTT control worked **without** any
> interaction with it, but if your unit won't wake after the switchover, this is the thing to look at.

### Topics and messages

Two topics, both keyed by the device's own ID (its MAC, as hex):

| Topic | Direction | Purpose |
|---|---|---|
| `/hentronix/dev/<DEVICE_ID>` | device → broker | **telemetry / status** (battery, door state, uptime…) |
| `/hentronix/app/<DEVICE_ID>` | broker → device | **commands / desired state**, delivered **retained** |

Payloads are small **JSON** objects of short keys (e.g. a device‑ID field, a monotonically increasing
uptime counter, and several state/command/setting fields). The exact keys are easy to read off your own
capture — and because the device is a sleepy subscriber, **commands are published `retained`** so the
broker hands the latest one over the instant the device next connects.

### Sleep, wake, and the app "session"

The ~16 s check‑ins are the device's **idle** behaviour — it just collects whatever command is waiting,
reports status, and drops the connection. Opening the unit in the app starts a **session**, and a local
takeover has to copy all three parts of it:

1. **Wake** — the app sends a "start session" command. The device then **holds one connection open**
   (an on‑board LED comes on) instead of dropping it every 16 s.
2. **Heartbeat** — while you're on the device's screen the app re‑sends a keep‑alive command every few
   seconds, and the device answers each with status. This is what keeps it awake.
3. **Sleep** — leaving the screen sends an "end session" command and the device returns to the idle 16 s
   pattern.

**This matters for battery life.** Because commands are **retained**, if your broker is left holding a
*wake* command the device re‑wakes on every check‑in and never sleeps — draining the battery. Home
Assistant must always leave a **sleep/idle** command as the retained value when it isn't actively
controlling the door (see [03](03-home-assistant-takeover.md)).

> Your unit's **broker IP, MQTT username/password and device ID are specific to you** and are captured
> from your own device — see [02](02-find-your-devices-broker.md). They are intentionally not published
> here.

## Why this is good news for a local takeover

- **No DNS** to spoof and **no TLS/pinning** to defeat — the simplest possible interception.
- The device targets a **fixed IP**, so one router rule can send that destination to a **local broker**,
  and it keeps working even after the real cloud host disappears.
- The device keeps its **own firmware, login and topics** — nothing is flashed or rewired.

The method to capture your own values is in [02](02-find-your-devices-broker.md); the takeover build is
in [03](03-home-assistant-takeover.md).
