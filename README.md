# Keeping a Hentronix WiFi coop door working with Home Assistant

Hentronix made WiFi‑connected automatic chicken‑coop door openers. The company has **ceased trading in
the UK**, and the "smart" side of the product depends on a **cloud MQTT broker** that nobody is clearly
answerable for any more. If that server goes offline, the phone app, remote control, schedule changes
and battery/status readings stop working — even though the door hardware is fine.

This repo documents how a Hentronix WiFi opener talks to its cloud, and a **no‑hardware‑change way to
take over control locally** with [Home Assistant](https://www.home-assistant.io/), so the door keeps
working for good — privately, with no dependence on a defunct company's servers.

> **For owners of their own hardware.** This is a right‑to‑repair / local‑control guide for a device
> **you own**. Every value that matters (your broker address, your login, your device ID) you read off
> **your own unit** — see [docs/02](docs/02-find-your-devices-broker.md). Don't connect to the shared
> cloud broker to poke around, and never touch a device you don't own.

## The short version

- The opener is a sleepy battery/solar **ESP‑class WiFi device**. Its daily **dawn/dusk schedule runs
  locally** on its own clock (RTC) + battery, so that part keeps working with or without the internet.
- For remote control it **dials out to a cloud MQTT broker** and relays messages with the phone app
  through it. On the units seen so far this is **plain, unencrypted MQTT (port 1883)** aimed at a
  **hardcoded IP address** — no DNS, no TLS, no certificate pinning.
- Because it's plaintext to a fixed address, you can **redirect it to your own MQTT broker at home**
  with a single **destination‑NAT (DNAT) rule on your router**. The opener keeps its firmware, login and
  topics — it just lands on your broker. Home Assistant then becomes the brain.
- The device is a **sleepy subscriber**: the app *wakes* it, keeps it awake with a heartbeat, then
  *sleeps* it. A local takeover has to copy that — and must always leave a **sleep** command as the
  retained value, or the door never sleeps and the battery drains.

## What's here

| Doc | What it covers |
|---|---|
| [docs/01-how-it-works.md](docs/01-how-it-works.md) | The opener's architecture and MQTT control model |
| [docs/02-find-your-devices-broker.md](docs/02-find-your-devices-broker.md) | How to capture **your own** device's broker IP, login and topics (passively) |
| [docs/03-home-assistant-takeover.md](docs/03-home-assistant-takeover.md) | The takeover: local broker + DNAT redirect + Home Assistant, step by step |
| [network-traffic-route.html](network-traffic-route.html) | A one‑page visual of the before/after traffic route (open in a browser) |

## What is deliberately NOT here

To avoid enabling access to the shared cloud broker or to other people's devices, this repo does **not**
publish: the live broker IP, the shared MQTT username/password that ships in the firmware, or any real
device ID. You obtain those for your own unit by capturing your own traffic
([docs/02](docs/02-find-your-devices-broker.md)). Placeholders like `<BROKER_IP>`, `<MQTT_USER>`,
`<MQTT_PASS>` and `<DEVICE_ID>` mark where your captured values go.

## Safety / ethics

- Do this only for **hardware you own**.
- Everything in the discovery phase is **passive observation** of your own device's traffic — no
  commands are sent to the door.
- The **first real open/close through Home Assistant should be a deliberate, supervised test with eyes
  on the coop.** Don't automate open/close until you've confirmed the command format from your own
  device.
- Mind the welfare angle: a coop door is a living‑animal safety device. Keep the door's own dawn/dusk
  schedule as the fallback, and don't leave it in a state where a bug could trap or expose the birds.

## Status

**End‑to‑end tested on one Hentronix WiFi unit (Oct 2026):** passive capture → local Mosquitto broker →
router DNAT redirect → **wake, live status (door position, battery, light, temperature) and open/close
confirmed moving the physical door**, then back to sleep — with the vendor cloud bypassed. Exact JSON
fields and command codes may differ between units and firmware, so treat the specifics as a **template to
confirm against your own capture**. PRs with other units' findings are welcome.

*Not affiliated with Hentronix. "Hentronix" is used only to identify the hardware this concerns.*
