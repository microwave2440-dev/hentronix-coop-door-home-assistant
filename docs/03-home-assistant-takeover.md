# 03 — The takeover: local broker + DNAT redirect + Home Assistant

Goal: the opener keeps its firmware but talks to **your** MQTT broker instead of the vendor cloud, and
**Home Assistant** becomes the control + monitoring brain — fully local, no company and no internet
dependency. No hardware change.

This assumes you've captured your own `<BROKER_IP>`, `<MQTT_USER>`, `<MQTT_PASS>`, `<DEVICE_ID>`, topics
and command format from [02](02-find-your-devices-broker.md). Order matters: **learn the command format
from the real app first** (step 0), because once the redirect is in place the vendor app can no longer
reach the door.

## Step 0 — Capture the command format (while the vendor cloud is still up)
See the end of [02](02-find-your-devices-broker.md). Operate your own door from the real app — open,
close, each setting once — and record the `/hentronix/app/<DEVICE_ID>` messages. This is your only clean
source of the command JSON, and it needs the vendor cloud alive, so do it before switching over.

## Step 1 — Stand up a local broker + Home Assistant
On an always‑on box on your network (give it a **static IP**, call it `<HA_IP>`):

- Install **Home Assistant** and the **Mosquitto broker** add‑on.
- Configure Mosquitto to accept the opener's login — either add a user matching the device's captured
  `<MQTT_USER>`/`<MQTT_PASS>`, or run a listener that allows it. Keep **retained** messages (default).
- It must accept the device's topics, e.g. `/hentronix/#`.

## Step 2 — Build the Home Assistant entities (before switching over)
Using what you captured:

- **Sensors** from `/hentronix/dev/<DEVICE_ID>` (parse the JSON): battery voltage, door state, uptime,
  signal, etc.
- **A `cover`** (and/or buttons/numbers for settings) that **publishes the captured command JSON** to
  `/hentronix/app/<DEVICE_ID>`. Use only values you've seen the real app send — never guesses.
- **Handle the wake/sleep session** (see the session model in [01](01-how-it-works.md)). The device only
  acts while awake, so a robust `cover` should: publish **wake** (retained) → wait for the device to
  report an open session (or its settings/status to arrive) → send the **move** command → watch the
  door‑state field reach the target → publish **sleep** (retained). Send a **heartbeat** on a short timer
  while you intend to stay connected. **Always finish by leaving a sleep/idle command as the retained
  value, with a timeout fallback** — if a wake command is left retained, the door never sleeps and the
  battery drains.

You can build and test all of this against the local broker before pointing the door at it.

## Step 3 — Redirect the broker IP to your broker (router DNAT)
The opener connects to a **fixed IP**, so intercept that destination at the router and send it to
`<HA_IP>:1883`. The device can't tell the difference — same IP from its side, same login, same topics.

**UniFi example** (UniFi Network ≥ 9 has NAT rules in the UI): a **Destination NAT**, pre‑routing,
match **source = `<DOOR_IP>`**, **destination = `<BROKER_IP>`**, **dest‑port `1883`**, translated to
`<HA_IP>:1883`; add a firewall allow for the door's VLAN → `<HA_IP>:1883`.

**Generic `iptables`/`nftables` equivalent on a Linux gateway:**
```
iptables -t nat -A PREROUTING -s <DOOR_IP> -d <BROKER_IP> -p tcp --dport 1883 \
  -j DNAT --to-destination <HA_IP>:1883
```
- If `<HA_IP>` is on the **same subnet/VLAN as the door**, also add a **masquerade (SNAT)** for that
  traffic, or the broker replies straight to the door, bypasses the gateway, and the connection breaks:
  ```
  iptables -t nat -A POSTROUTING -s <DOOR_IP> -d <HA_IP> -p tcp --dport 1883 -j MASQUERADE
  ```
  On a **different** VLAN, replies already route back through the gateway and no SNAT is needed.
- On appliance firmwares (UniFi OS etc.), persisting hand‑rolled rules across reboots is the only fiddly
  part — prefer the UI rule where one exists.

Because the rule owns that destination for the device, it keeps working even after the vendor's host
disappears or its IP is reassigned.

### Alternative if there's no DNS and you'd rather not DNAT
If your capture *did* show a **hostname** (DNS) instead of a hardcoded IP, you can skip DNAT and simply
override that hostname to `<HA_IP>` in your router's local DNS. On the units seen so far it was a
hardcoded IP, so DNAT is the route.

## Step 4 — Supervised cutover, then retire the app
- First open/close through Home Assistant: **deliberate, with someone watching the coop.** If anything
  misbehaves, removing the DNAT rule puts the door back on the old broker (while it still exists), and
  the door's own dawn/dusk schedule keeps running regardless.
- Once telemetry and commands are confirmed, retire the vendor app. HA now owns monitoring and control,
  locally.
- **Optional during transition:** instead of a hard DNAT, bridge your Mosquitto to the real broker (same
  creds) so both app and HA work while you validate, then drop the bridge. Only useful while the vendor
  cloud is still alive.

## Fallbacks
- **Reflash the ESP to open firmware** (ESPHome/Tasmota) — only needed if the device uses TLS with
  certificate pinning, or if you want to drop the cloud shape entirely. Same board, new firmware; keep a
  backup of the original flash. This changes firmware (not hardware), so it's a last resort.
- **DNS override** — only applicable if your device resolves a hostname (see step 3 alternative).

## Keep the welfare fallback
A coop door protects living animals. Leave the device's **local dawn/dusk schedule** as the safety net,
and don't deploy automations that could trap or expose the birds if Home Assistant, the broker or the
server has a bad day.
