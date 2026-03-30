# ADR-002: BLE Beacon Scanning Approach for Hotel Room Cleaning Tracker

| Field | Detail |
|---|---|
| **ADR Number** | ADR-002 |
| **Title** | BLE Beacon Scanning Approach for Hotel Room Cleaning Tracker |
| **Status** | Accepted |
| **Date** | 2026-03-30 |
| **Deciders** | Engineering Team, Hotel Operations |
| **Reviewed By** | Infrastructure Lead, Product Owner |

---

## Context

The hotel room cleaning tracker requires a mechanism to detect which room a cleaner is currently in, based on BLE signals emitted by a wearable tag (Minew C10 card badge) worn by each cleaner on their lanyard. The system must reliably determine room-level presence and report this to the ASP.NET Core backend so the 10-minute dwell timer logic can operate.

Two approaches were evaluated for how BLE scanning is performed and how location data reaches the .NET backend:

- **Option A** — A .NET MAUI mobile app running on the cleaner's smartphone scans for fixed room beacons and reports the detected room to the ASP.NET Core API
- **Option B** — A Raspberry Pi installed per floor scans for the cleaner's wearable BLE tag and publishes location events to the MQTT broker, which the ASP.NET Core backend subscribes to

Both options integrate with the same ASP.NET Core backend and SQL Server/PostgreSQL database. The difference lies entirely in where the BLE scanning happens and what hardware the cleaner carries.

---

## Decision Drivers

- **Staff simplicity** — minimal friction for cleaning staff during their shift
- **Hardware dependency** — avoid reliance on cleaners' personal smartphones
- **Reliability** — location detection must work consistently across shift hours
- **No app maintenance** — reduce ongoing mobile app development and update overhead
- **Infrastructure fit** — align with existing decision to use Raspberry Pi for BLE scanning (see ADR-002 context)
- **Battery and device management** — minimise charging and device management burden on staff
- **Security** — avoid installing hotel software on personal devices
- **Scalability** — solution must work across multiple floors and properties

---

## Options Considered

### Option A — .NET MAUI Smartphone App

The cleaner carries a smartphone running a .NET MAUI application. The app uses the `Plugin.BLE` NuGet package to continuously scan for fixed BLE beacons installed in each hotel room. The app determines the current room based on the beacon with the strongest RSSI signal and posts the detected room ID to the ASP.NET Core REST API over the hotel Wi-Fi network. The dwell timer logic runs in the backend, and the room is marked clean after 10 continuous minutes in the same room.

**How it works:**
1. Fixed BLE beacons (e.g. Feasycom DA14531) are placed in each room
2. Cleaner opens the MAUI app at the start of their shift
3. App scans continuously for beacons and identifies the strongest signal
4. App posts `{ cleanerId, roomId }` to `/api/location` every 20–30 seconds
5. ASP.NET Core backend runs dwell timer and updates room status

**Pros:**
- Cleaners can receive push notifications, task lists, and shift updates on the same device
- No additional hardware per floor — scanning happens on the cleaner's existing smartphone
- Familiar interface — cleaners interact with a purpose-built app
- .NET MAUI allows a single codebase for both iOS and Android
- Direct REST API integration — no MQTT broker required for this path
- Real-time feedback to cleaner (e.g. "Room 401 marked clean!")

**Cons:**
- Requires cleaners to carry and use a smartphone during their shift
- Hotel must either provide smartphones or rely on cleaners' personal devices
- Installing hotel software on personal devices raises privacy concerns and BYOD policy complications
- App must run in the background continuously — risk of OS killing the app on battery-saving modes
- Requires ongoing mobile app development, testing, and deployment to app stores
- Background BLE scanning on iOS is significantly restricted by Apple — requires careful permission handling
- If the cleaner's phone battery dies mid-shift, all location tracking for that cleaner stops
- App version management across multiple devices adds operational overhead

---

### Option B — Raspberry Pi BLE Scanner ✅ SELECTED

A Raspberry Pi 4 Model B is installed in the network cabinet on each hotel floor. The Pi runs a Python script using `bluepy` that continuously scans for BLE signals from cleaner wearable tags (Minew C10 badge). The Pi detects the tag's signal alongside fixed room beacons to determine which room the cleaner is in, then publishes a location event to the Stackhero Mosquitto MQTT broker. The ASP.NET Core background worker subscribes to these MQTT topics using MQTTnet and runs the 10-minute dwell timer logic.

The cleaner wears only the Minew C10 card badge on their existing lanyard — no smartphone, no app, no charging.

**How it works:**
1. Minew C10 card beacons (wearable) are issued to each cleaner on their lanyard
2. Fixed room beacons (e.g. Feasycom DA14531) are placed in each room
3. Raspberry Pi on each floor scans continuously for both tag and room beacon signals
4. Pi calculates closest room based on RSSI and publishes to `hotel/cleaner/<id>/location`
5. ASP.NET Core MqttWorker subscribes and runs dwell timer
6. Room marked clean after 10 minutes; dashboard and PMS updated

**Pros:**
- No smartphone required — cleaner only wears a card badge on their existing lanyard
- No mobile app to develop, maintain, or push updates to
- No BYOD or privacy concerns — hotel infrastructure only
- Tag battery life on Minew C10 is up to 1 year — negligible maintenance overhead
- Raspberry Pi operates 24/7 unattended in the network cabinet
- Background scanning is not OS-restricted — Pi scans continuously without interruption
- Single Pi covers an entire floor — cost-effective infrastructure
- Consistent and reliable scanning regardless of cleaner's personal device status
- Easily extended to multiple floors by adding one Pi per floor
- Decoupled from cleaner behaviour — tracking works even if cleaner forgets to open an app

**Cons:**
- Requires Raspberry Pi hardware installation per floor (~$135–$150 AUD each)
- Requires basic Python scripting knowledge to configure and maintain the scanner script
- Pi must be connected to the hotel LAN and have network access to the MQTT broker
- If the Pi fails, an entire floor loses location tracking until the Pi is replaced or restarted
- RSSI-based room detection on the Pi can be affected by signal interference between adjacent rooms
- No direct feedback loop to cleaner — a separate notification mechanism is needed for task completion alerts

---

## Comparison Summary

| Criteria | Option A: .NET MAUI App | Option B: Raspberry Pi Scanner |
|---|---|---|
| Cleaner carries | Smartphone + app | Card badge on lanyard only |
| Hardware cost per floor | $0 (uses existing phone) | ~$135–$150 AUD (Pi + accessories) |
| App development required | Yes — iOS + Android MAUI app | No |
| App maintenance required | Yes — ongoing updates | No |
| Background scanning reliability | Low on iOS, moderate on Android | High — uninterrupted |
| BYOD / privacy concern | Yes | No |
| Battery dependency | Cleaner's phone battery | Tag: ~1 year, Pi: mains powered |
| Staff friction | Medium — must carry phone, keep app open | Very low — badge on existing lanyard |
| Failure impact | Per cleaner (one phone dies) | Per floor (one Pi offline) |
| Real-time cleaner feedback | Yes (push notifications) | Requires separate notification channel |
| .NET MAUI skill required | Yes | No |
| Python skill required | No | Basic |
| MQTT broker required | No (REST API sufficient) | Yes (Stackhero, per ADR-003) |
| Scales to multiple properties | Per-device app deployment | Per-floor Pi provisioning |

---

## Decision

**Option B — Raspberry Pi BLE scanner is selected.**

The Raspberry Pi approach is the operationally superior choice for a hotel environment. Cleaners should not be required to carry a smartphone, keep an app running in the background, or manage a separate device during a physically demanding shift. The Minew C10 card badge requires no interaction — it is clipped to the cleaner's existing lanyard and forgotten. This eliminates the single largest source of tracking failure in Option A: human behaviour.

The Raspberry Pi scans continuously without OS restrictions, operates 24/7 from the hotel's mains power, and requires no ongoing software updates once deployed. The cost of one Pi per floor (~$135–$150 AUD) is justified by the elimination of mobile app development and maintenance costs, BYOD complications, and the operational risk of tracking failures due to phone battery or app state issues.

Option A is recommended only if the hotel intends to build a broader mobile experience for cleaners — for example, a task management app with shift scheduling and supervisor communication — in which case the BLE scanning can be incorporated into the same app as a secondary benefit.

---

## Architecture Overview

```
Minew C10 badge (cleaner wears on lanyard)
        │  BLE 5.0 broadcast signal
        ▼
Feasycom DA14531 beacons (fixed, one per room)
        │  RSSI detected by Pi on same floor
        ▼
Raspberry Pi 4 Model B (one per floor)
running: scanner.py (bluepy + paho-mqtt)
        │  MQTT publish — TLS port 8883
        │  topic: hotel/cleaner/<id>/location
        ▼
Stackhero Mosquitto broker (cloud, per ADR-003)
        │  MQTTnet subscribe
        ▼
ASP.NET Core background worker
(MqttWorker → DwellTimerService)
        │  10-min dwell timer per cleaner
        ▼
SQL Server / PostgreSQL
(room status: dirty | in_progress | clean)
        │
        ▼
Dashboard + PMS integration
```

**Technology stack per layer:**

| Layer | Technology |
|---|---|
| Cleaner wearable | Minew C10 card beacon (BLE 5.0) |
| Room reference beacons | Feasycom DA14531 (fixed, BLE 5.1) |
| Floor scanner | Raspberry Pi 4 Model B 4GB |
| Scanner language | Python 3 with `bluepy` and `paho-mqtt` |
| Message transport | MQTT over TLS (Stackhero, per ADR-003) |
| Backend subscriber | ASP.NET Core `BackgroundService` with MQTTnet |
| Timer logic | `DwellTimerService` (C#, `System.Threading.Timer`) |
| Persistence | Entity Framework Core + SQL Server / PostgreSQL |
| Dashboard | ASP.NET Core Razor Pages or Blazor Server |

---

## Hardware Shopping List (Per Floor — Melbourne, Australia)

| Component | Product | Supplier | Est. Cost (AUD) |
|---|---|---|---|
| Floor scanner | Raspberry Pi 4 Model B 4GB | Pi Australia (piaustralia.com.au) | ~$85 |
| Power supply | Official Raspberry Pi USB-C 5.1V 3A | Pi Australia | ~$15 |
| Storage | 32GB MicroSD (preloaded with Raspberry Pi OS) | Core Electronics | ~$20 |
| Enclosure | Official Raspberry Pi 4 case | Pi Australia | ~$15 |
| Optional BLE dongle | ASUS USB-BT500 (if Pi in metal cabinet) | Amazon AU | ~$25 |
| Cleaner wearable tag | Minew C10 card beacon (per cleaner) | AliExpress / minew.com | ~$20–$25 |
| Room reference beacon | Feasycom DA14531 (per room) | Amazon AU (B092V9BNMD) | ~$20–$25 |

**Estimated cost per floor (Pi + accessories):** ~$135–$160 AUD
**Estimated cost per cleaner (wearable tag):** ~$20–$25 AUD
**Estimated cost per room (reference beacon):** ~$20–$25 AUD

---

## Consequences

**Positive:**
- Zero friction for cleaning staff — badge is passive, no app interaction required
- Continuous, uninterrupted BLE scanning not subject to mobile OS restrictions
- No mobile app development or app store deployment required
- Tracking works regardless of cleaner's personal device status or battery level
- Single Pi per floor is cost-effective and straightforward to replace if faulty
- Clean separation of concerns — scanning infrastructure is independent of cleaner devices

**Negative:**
- Upfront hardware cost per floor for Raspberry Pi and accessories
- Pi failure takes out tracking for an entire floor until replaced or rebooted
- No direct real-time feedback to cleaner — a separate mechanism (e.g. SMS, push via supervisor tablet) is needed for task completion notifications
- Requires basic Python scripting capability on the team to configure and update scanner scripts

**Risks and Mitigations:**

| Risk | Likelihood | Mitigation |
|---|---|---|
| Raspberry Pi hardware failure | Low | Keep one spare Pi on-site; Docker + auto-restart ensures the scanner script recovers from crashes automatically |
| RSSI interference between adjacent rooms | Medium | Set minimum RSSI threshold per room; require 3+ consecutive readings before committing room assignment |
| Hotel LAN outage disconnects Pi from MQTT broker | Low | Pi queues messages locally using paho-mqtt persistent session; messages delivered on reconnect |
| Cleaner forgets to wear badge | Low | Supervisor dashboard flags cleaners with no recent location events; issue replacement badges |
| Pi not visible to MQTT broker through hotel firewall | Low | Pi connects outbound to Stackhero on port 8883 — no inbound port forwarding required |
| bluepy library deprecated or unsupported | Low | Replace with `bleak` (pure Python, async BLE library) — drop-in replacement with minor script changes |

---

## Links and References

- Minew C10 card beacon: https://www.minew.com/product/c10-card-beacon/
- Feasycom DA14531 beacon (Amazon AU): https://www.amazon.com.au/dp/B092V9BNMD
- Raspberry Pi 4 Model B (Pi Australia): https://piaustralia.com.au/products/raspberry-pi-4
- Core Electronics (Australian Pi supplier): https://core-electronics.com.au
- bluepy Python BLE library: https://github.com/IanHarvey/bluepy
- bleak Python BLE library (modern alternative): https://github.com/hbldh/bleak
- paho-mqtt Python MQTT client: https://github.com/eclipse/paho.mqtt.python
- MQTTnet .NET library: https://github.com/dotnet/MQTTnet
- Plugin.BLE NuGet (.NET MAUI, Option A reference): https://github.com/dotnet-bluetooth-le/dotnet-bluetooth-le
- ADR-002: Indoor positioning technology decision (BLE Beacons selected)
- ADR-003: MQTT broker decision (Stackhero selected)

---

*ADR-002 | Status: Accepted | Last updated: 2026-03-30*
