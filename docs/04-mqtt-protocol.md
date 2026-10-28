# 04 — MQTT protocol reference (decoded)

> **Read this as a starting point, not gospel.** Everything below was **observed on one Hentronix WiFi
> unit (Oct 2026)**. Confirm the exact codes and fields against **your own capture** — they may differ
> between units and firmware. Doc [02](02-find-your-devices-broker.md) is how you capture your own
> values; this page is the reference to **compare them against**, not a substitute for capturing them.

This is the decoded shape of the plaintext MQTT traffic described in [01](01-how-it-works.md): the two
topics, the command frame the app sends, the command codes, the status fields the device reports, and —
most importantly — the **session model** that governs when the device will actually listen.

## Topics

Both topics are keyed by the device's own ID (its MAC, as hex — your `<DEVICE_ID>`):

| Topic | Direction | Purpose |
|---|---|---|
| `/hentronix/dev/<DEVICE_ID>` | device → broker | status / telemetry |
| `/hentronix/app/<DEVICE_ID>` | broker → device | commands (delivered **retained**) |

## The command frame (app → device)

Commands are small **JSON** objects published to `/hentronix/app/<DEVICE_ID>`. The shape seen on the test
unit (placeholder values):

```json
{"D1":"<DEVICE_ID>","A2":"<SESSION_TOKEN>","C1":<command>,"V1":<value>,"O1":0,"C2":2,"W":<counter>}
```

- `D1` — the device ID, as in the topic.
- `A2` — the **session token** echoed back to the device (captured from its status while in a session).
- `C1` — the **command code** (see below).
- `V1` — the command's **value**, where the command takes one (e.g. the move direction).
- `W` — a **required** rolling counter (see "The `W` field is required" below).

### Command codes (`C1`)

| `C1` | Meaning | `V1` | Notes |
|---|---|---|---|
| `8` | **Wake / start session** | — | Device holds its connection open, an on‑board LED comes on, and it replies with a **full settings dump**. |
| `9` | **Heartbeat / keep‑alive** | — | The app sends this roughly **every ~3 s** while you're on the device's screen; the device answers each one with a status message. |
| `1` | **Move the door** | direction | On the unit tested, **`V1:4` = OPEN** and **`V1:1` = CLOSE**. |
| `41` | **Coop light / "2‑minute torch"** | — | Turns the light on; on the test unit it **timed out by itself after ~2 min**. |
| `21` | **Sleep / end session** | — | Returns the device to idle. This is also the value the broker should hold **retained when idle**. |

> **⚠️ Confirm the move direction on your own unit before relying on it.** `V1:4` = OPEN / `V1:1` = CLOSE
> is what *this* unit did — do **not** assume yours matches. Getting it backwards can **trap or expose the
> birds**. Confirm it with a deliberate, supervised test (door in view) before you automate anything.

### The `W` field is required

Every command the app sent carried a **`W`** field — a small, changing integer. **Commands published
without a `W` field were silently ignored and the door stayed asleep.** Its value didn't appear to be
validated (any changing small integer worked), but the **field had to be present**. A rolling counter
that mimics the app is fine. If your replayed commands do nothing, this is the first thing to check.

## Status fields (device → device topic)

Published as JSON on `/hentronix/dev/<DEVICE_ID>`. Labelled **confirmed**, **likely** or **unconfirmed** —
confirm your own either way.

| Field | Meaning | Status |
|---|---|---|
| `D1` | Device ID | confirmed |
| `A2` | **Session token** — echoed while in an app session, empty string `""` when idle | confirmed |
| `C1` | Echoes the command being answered; `9` in a normal idle status | confirmed |
| `C2` | `1` while in an app session, `0` when idle | confirmed |
| `D2` | **Door position** — `1` closed, `2` closing, `3` opening, `4` open | confirmed by watching the physical door (confirm your own) |
| `V3` | Battery voltage **now**, ×10 (e.g. `150` = 15.0 V) | confirmed (matched the app) |
| `V2` | Battery voltage **last night**, ×10 | likely |
| `T1` | Temperature, ×10 (°C) | confirmed (matched the app) |
| `L4` | Coop light — `0` off, `1` on | confirmed |
| `R2` | Uptime in seconds, increments ~1/s | confirmed |
| `W` | A small, constantly changing integer (a counter or signal value) | purpose unconfirmed — but **required on commands** (see above) |

### Settings‑dump‑only fields

After a `C1:8` wake, the device replies with a larger **settings dump** containing many more keys.
Meanings are mostly **unconfirmed** — treat this as a group to decode from your own unit:

```
S5  S6  S8  S9  O1  O2  O3  O4  C3  C4  C5  L1  L2  L3  L5  L6
N1  D3  A1  P1  B1  V4  OT  AO  AC  CN  DN
```

A few were matched to settings shown in the app:

| Field | Meaning (matched to app) |
|---|---|
| `C4` | Minutes to close after dusk |
| `O4` / `C5` | Open / close motor power (`2` = medium) |
| `L3` | Light intensity, % |
| `B1` | Battery type |

## The session model

This is the key thing that saves you time. The device is a **sleepy subscriber** — it does not sit
listening. It cycles between an **idle** pattern and a **responsive** session:

1. **Idle.** The device reconnects roughly every **~16 s**, collects whatever command is waiting
   (**retained**), publishes **one** status message, and drops the connection.
2. **Wake (`C1:8`).** The device holds **one** connection open and switches to responsive mode.
3. **While awake**, the app sends `C1:9` every **~3 s** and the device answers each — **this is what keeps
   it awake**.
4. **Sleep (`C1:21`).** The device returns to the idle ~16 s pattern.

### Retained vs live

- **Open / close / torch** are sent **LIVE** (not retained), while the device is awake.
- **Wake and sleep** are sent **RETAINED** — a sleeping device only reads the retained value on its next
  check‑in, so a retained wake is how you get an idle device to come up.

### ⚠️ Battery warning

Because commands are **retained**, you must **always leave `C1:21` (sleep) as the retained value when
idle**, with a timeout fallback. If a **wake** command is left retained, the device **re‑wakes on every
check‑in and never sleeps — draining the battery.**

## Recommended control sequence

For a Home Assistant `cover` (or similar), a robust move looks like:

1. Publish **wake** — `C1:8`, **retained**, with a `W` value.
2. **Wait** for the device to report a session (`C2:1`) or its settings dump to arrive.
3. Send the **move** command — `C1:1` with the right `V1`, **live** (not retained).
4. **Watch `D2`** reach the target position (`4` open / `1` closed).
5. Publish **sleep** — `C1:21`, **retained**.

Send `C1:9` (heartbeat) on a short timer for as long as you intend to stay connected. And — see the
battery warning — make sure the resting retained value is always **sleep**.

See [03](03-home-assistant-takeover.md) for building this into Home Assistant, and
[01](01-how-it-works.md) for the architecture behind it.
