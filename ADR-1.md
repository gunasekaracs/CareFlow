# ADR-002: Indoor Positioning Technology for Hotel Room Cleaning Tracker

| Field | Detail |
|---|---|
| **ADR Number** | ADR-001 |
| **Title** | Indoor Positioning Technology for Hotel Room Cleaning Tracker |
| **Status** | Accepted |
| **Date** | 2026-03-26 |
| **Deciders** | Engineering Team, Hotel Operations |
| **Reviewed By** | Infrastructure Lead, Product Owner |
| **Revision** | 1.1 — Blueiot AoA added as Option 4 |

---

## Context

The hotel requires an automated system to track when a cleaner has spent at least **10 continuous minutes inside a single room**, at which point the room is automatically marked as cleaned. This eliminates manual check-off processes and provides real-time visibility to supervisors via a dashboard integrated with the Property Management System (PMS).

The system must:
- Identify which room a cleaner is currently in
- Track continuous dwell time per room
- Trigger a "room cleaned" event after 10 uninterrupted minutes
- Reset the timer if the cleaner leaves the room before the threshold is met
- Integrate with a Node.js backend and existing hotel PMS

Four technology options were evaluated for the indoor positioning layer.

---

## Decision Drivers

- **Room-level accuracy** — must reliably distinguish adjacent rooms
- **Infrastructure cost** — hardware cost per room and total deployment cost
- **Setup simplicity** — speed of deployment and ease of configuration
- **Staff experience** — minimal friction for cleaning staff day-to-day
- **Integration simplicity** — ease of connecting to Node.js backend
- **Reliability** — resistance to signal interference and environmental factors
- **Regulatory compliance** — no special licensing or approvals required
- **Scalability** — suitable for multi-floor, multi-property deployment
- **Future extensibility** — potential to support additional use cases beyond cleaning

---

## Options Considered

### Option 1 — BLE (Bluetooth Low Energy) Beacons ✅ SELECTED

BLE beacons are small, battery-powered transmitters placed in each room. The cleaner carries a smartphone running a lightweight app that continuously scans for nearby beacon signals. Signal strength (RSSI) from each beacon is used to determine the cleaner's current room — the beacon with the strongest signal wins. Dwell time is tracked in the Node.js backend via MQTT or REST API. When the cleaner remains in a single room for 10 uninterrupted minutes, the backend automatically marks the room as cleaned.

**Pros:**
- Lowest hardware cost (~$10–$30 per room beacon)
- Works with standard smartphones — no dedicated hardware tag required for staff
- Lowest setup complexity of all options; beacons are plug-and-play
- Easy Node.js integration via MQTT or REST API
- No regulatory requirements
- Large, mature ecosystem with many off-the-shelf solutions (Estimote, Kontakt.io, Minew)
- Cleaners can receive push notifications and view their task list on the same device

**Cons:**
- No mesh networking — requires a Wi-Fi gateway within range of all areas
- Beacon battery replacement required every 6–18 months
- Signal bleed through thin hotel walls can occasionally cause false room assignments
- Accuracy (1–3m) is RSSI-based and susceptible to environmental interference
- Less suitable for large open-plan suites without careful beacon placement

---

### Option 2 — Zigbee

Zigbee is a low-power mesh radio protocol operating on 2.4GHz. Each room is fitted with a mains-powered Zigbee router node (which can double as a smart light switch or plug). The cleaner wears a small, long-life Zigbee end-device tag. The tag's signal is picked up by nearby router nodes and relayed through the self-healing mesh network to a Zigbee coordinator on a Raspberry Pi. Zigbee2MQTT translates device events into MQTT messages that the Node.js backend subscribes to.

**Pros:**
- Self-healing mesh network — devices relay signals hop-by-hop, no gateway gaps
- Excellent battery life on dedicated tags (1–3 years)
- Integrates with existing Zigbee hotel infrastructure (smart lighting, door locks, thermostats)
- Strong Node.js integration via Zigbee2MQTT → Mosquitto MQTT broker
- Channel hopping avoids interference with Wi-Fi and BLE

**Cons:**
- Requires a dedicated Zigbee tag for each cleaner — no smartphone app option
- Higher hardware cost than BLE (~$20–$40 per room node)
- Moderate setup complexity — requires Zigbee2MQTT, MQTT broker, and Raspberry Pi configuration
- Staff must carry and manage an additional physical device
- Room-level accuracy (2–5m) slightly lower than BLE

---

### Option 3 — GPS (Including Indoor GPS / Pseudolite)

Standard GPS relies on line-of-sight to satellites and does not function reliably indoors. An indoor GPS variant using pseudolites (ground-based transmitters that emit GPS-like signals) was also considered. Pseudolites are used in mining, military, and large warehouse settings to provide GPS-style positioning without satellite dependency.

**Pros:**
- Familiar technology concept
- Standard GPS works well for outdoor campus/resort geofencing
- Pseudolite systems can theoretically achieve room-level accuracy indoors

**Cons:**
- Standard GPS is entirely unsuitable indoors — accuracy degrades to 10–50m+ inside buildings
- Pseudolite systems cost $100,000–$500,000+ to deploy in a multi-floor hotel
- Pseudolites require spectrum licensing in Australia (ACMA)
- Standard smartphones cannot receive pseudolite signals — requires specialised hardware receivers
- Extremely high RF engineering complexity to manage multipath signal reflections
- No viable path to Node.js integration without a proprietary SDK
- Not used in any commercial hotel operations context globally

---

### Option 4 — Blueiot (Bluetooth AoA — Angle of Arrival)

Blueiot is a Real-Time Location System (RTLS) built on **Bluetooth 5.1 AoA (Angle of Arrival)** technology — a fundamentally different positioning method to standard BLE RSSI beacons. Rather than estimating proximity from signal strength, AoA anchor devices use a built-in multi-antenna array to measure the precise *angle* at which the cleaner's tag signal arrives. Multiple anchors triangulate to determine an exact two-dimensional or three-dimensional position. Blueiot provides the full stack: AoA anchors, personnel tags, a positioning engine (local or cloud), and an open API for integration.

The positioning engine exposes a REST API and WebSocket stream, making it straightforward to consume real-time location data in a Node.js backend without requiring an MQTT broker or Zigbee middleware layer.

**How it works:**
1. AoA anchors are mounted on ceilings — one anchor can cover an entire floor or wing
2. Each cleaner wears a small Bluetooth 5.1 tag that transmits a Constant Tone Extension (CTE) signal
3. Each anchor's antenna array calculates the angle of arrival and sends data to the Blueiot positioning engine
4. The engine resolves x/y coordinates at 0.1–0.3m accuracy and pushes updates via REST API or WebSocket
5. The Node.js backend consumes the location stream, maps coordinates to room IDs, and runs dwell timer logic

**Pros:**
- Highest accuracy of all options — 0.1–0.3m (sub-metre), compared to 1–3m for standard BLE RSSI
- Fewer anchors needed — a single base station can cover one floor, reducing installation points significantly
- Tag battery life exceeds 5 years — lowest ongoing maintenance overhead of all options
- Strong anti-interference capability — phase difference direction finding resists multipath reflections from walls, glass, and metal
- Compatible with all Bluetooth 4.0+ devices and standard smartphones
- Open REST API and WebSocket output — clean Node.js integration without additional middleware
- Purpose-built for personnel tracking in complex indoor environments (healthcare, hospitality, logistics)
- Highly scalable — a single anchor handles up to 500 concurrent tags
- Future use cases unlocked: precise asset tracking, occupancy analytics, visitor navigation, emergency mustering

**Cons:**
- Higher upfront system cost than standard BLE beacons — anchor hardware and Blueiot platform licensing
- Requires dedicated Blueiot tags for staff (though compatible with standard Bluetooth devices)
- Moderate setup complexity — anchor placement requires site survey to optimise coverage angles
- Blueiot is a specialised vendor; less community support than the BLE beacon ecosystem
- Slight overkill for basic room-level detection — sub-metre precision exceeds the minimum requirement
- Hardware ships from Beijing; lead time approximately 1–2 weeks to Australia

---

## Comparison Summary

| Criteria | Option 1: BLE Beacons | Option 2: Zigbee | Option 3: GPS | Option 4: Blueiot AoA |
|---|---|---|---|---|
| Positioning method | RSSI signal strength | RSSI signal strength | Satellite triangulation | Angle of Arrival (AoA) |
| Room-level accuracy | Good (1–3m) | Good (2–5m) | Not viable indoors | Excellent (0.1–0.3m) |
| Standard smartphone support | Yes | No — dedicated tag | Partial (hybrid only) | Yes (BT 4.0+) |
| Tag / device battery life | 6–18 months (beacon) | 1–3 years (tag) | Hours (active GPS) | 5+ years (tag) |
| Anchors / beacons needed | 1 per room | 1 router per room | N/A | 1 per floor / wing |
| Mesh networking | No | Yes — self-healing | No | No (star topology) |
| Hardware cost (100 rooms) | $1,500–$3,000 | $2,000–$4,000 | $100,000–$500,000+ | $5,000–$15,000 (est.) |
| Setup complexity | Low | Moderate | Very high | Moderate |
| Existing hotel ecosystem fit | Moderate | High | None | Moderate |
| Regulatory requirements | None | None | Spectrum licence (AU) | None |
| Indoor reliability | High | High | Very poor | Very high |
| Node.js integration | MQTT / REST API | MQTT via Zigbee2MQTT | Proprietary SDK only | REST API / WebSocket |
| Future extensibility | Low | Moderate | None | High |
| Staff device required | Smartphone (existing) | Dedicated Zigbee tag | Specialised receiver | Small BT 5.1 tag |

---

## Decision

**Option 1 — BLE Beacons is selected.**

BLE beacons offer the best combination of low cost, fast deployment, and minimal friction for cleaning staff at this stage. Because cleaners use their existing smartphones rather than a dedicated hardware tag, there is no additional device to distribute, charge, or replace. The beacon hardware is inexpensive, widely available in Australia, and requires no specialist configuration. Integration with the Node.js backend via MQTT is straightforward and well-documented.

Blueiot AoA (Option 4) is acknowledged as a technically superior solution — particularly its sub-metre accuracy, per-floor anchor coverage, and 5-year tag battery life — and is the **recommended upgrade path** if the hotel later expands the system to support asset tracking, visitor navigation, or multi-property operations. Its higher upfront cost and dedicated tag requirement make it less optimal for an initial deployment focused solely on room cleaning verification.

Zigbee is preferred over BLE if the hotel already has Zigbee smart infrastructure deployed. GPS is rejected outright due to fundamental indoor limitations.

---

## Architecture Overview

```
BLE Beacons (one per room, battery-powered)
        │ Bluetooth RSSI signal
        ▼
Cleaner's Smartphone (iOS or Android app)
        │ HTTP / MQTT over Wi-Fi
        ▼
Wi-Fi Gateway (existing hotel AP infrastructure)
        │
        ▼
Node.js Backend (dwell timer logic, room state machine)
        │
        ▼
Database + Dashboard / PMS Integration
```

**MQTT Topic Structure:**
- `hotel/cleaner/<cleaner_id>/location` — smartphone publishes detected room ID
- `hotel/rooms/status` — Node.js publishes cleaned status updates
- PMS webhook fires on room status change to `clean`

**Dwell Timer Logic:**
1. Smartphone app continuously scans BLE beacons and identifies the strongest RSSI
2. Detected room ID is sent to the Node.js backend every 15–30 seconds
3. Backend starts a 10-minute countdown timer for the cleaner/room pair
4. If the detected room changes, the timer resets immediately
5. A 45-second grace period prevents resets from brief corridor crossings
6. On timer completion, a "room cleaned" event is persisted and broadcast to the dashboard

---

## Consequences

**Positive:**
- Fastest path to deployment — beacons can be placed and working within hours
- No new hardware for cleaning staff — uses smartphones they already carry
- Supervisor dashboard provides real-time visibility across all floors
- PMS auto-update removes manual room-ready check-off entirely
- Low ongoing maintenance cost — beacons only need battery replacement every 12–18 months
- Clear upgrade path to Blueiot AoA for higher precision use cases in future

**Negative:**
- Cleaning staff must have their smartphones on them and the app running during shifts
- Beacon battery replacement creates periodic maintenance overhead across all rooms
- Thin walls in older hotel buildings may require RSSI threshold tuning to prevent false room assignments
- No mesh redundancy — if the Wi-Fi gateway in an area goes offline, location reporting pauses for that zone

**Risks and Mitigations:**

| Risk | Likelihood | Mitigation |
|---|---|---|
| Signal bleed between adjacent rooms | Medium | Set minimum RSSI threshold; require 3+ consecutive matching readings before committing room assignment |
| Staff leaves phone at trolley | Low | Supervisor dashboard flags cleaner as inactive; push notification after 5 mins with no movement |
| Beacon battery dies mid-shift | Low | Monitor beacon health via Kontakt.io / Estimote cloud dashboard; replace on schedule |
| Wi-Fi outage in a zone | Low | App queues location events locally and syncs when connectivity resumes |
| App killed by phone OS in background | Medium | Use background BLE scanning permissions; test on both iOS and Android |

---

## Recommended Hardware (Australia)

### Option 1 — BLE Beacons (Selected)

| Component | Product | Supplier | Est. Cost |
|---|---|---|---|
| BLE beacons | Minew E8 or Kontakt.io S18 | AliExpress / kontakt.io | ~$15–$25 each |
| Beacon management | Kontakt.io Cloud or Estimote Cloud | kontakt.io / estimote.com | Free tier available |
| Staff smartphones | Existing Android / iOS devices | — | $0 (existing) |
| MQTT broker | Mosquitto (self-hosted) or HiveMQ Cloud | mosquitto.org / hivemq.com | Free tier available |

**Estimated total hardware cost for a 100-room hotel:** ~$1,500–$2,500 AUD

### Option 4 — Blueiot AoA (Future Upgrade Path)

| Component | Product | Supplier | Est. Cost |
|---|---|---|---|
| AoA anchors | Blueiot BA3000-T ceiling anchor | blueiot.com | ~$200–$400 each |
| Personnel tags | Blueiot BT110 wristband or card tag | blueiot.com | ~$30–$50 each |
| Positioning engine | Blueiot AoA Engine (local or cloud) | blueiot.com | Licence fee — contact vendor |
| Demo / POC kit | Blueiot Development Kit | blueiot.com | Available on request |

**Estimated total hardware cost for a 100-room hotel (Blueiot):** ~$5,000–$15,000 AUD (fewer anchors offset higher unit cost)

---

## Links and References

- Kontakt.io BLE beacons: https://kontakt.io
- Estimote BLE beacons: https://estimote.com
- Minew beacons: https://www.minewtech.com
- Blueiot AoA RTLS: https://www.blueiot.com
- Blueiot AoA technology overview: https://www.blueiot.com/technology
- Blueiot positioning engine API: https://www.blueiot.com/product/aoa-engine
- Mosquitto MQTT broker: https://mosquitto.org
- HiveMQ Cloud (managed MQTT): https://www.hivemq.com/mqtt-cloud-broker
- Node.js MQTT client library: https://github.com/mqttjs/MQTT.js

---

*ADR-001 v1.1 | Status: Accepted | Last updated: 2026-03-26*
