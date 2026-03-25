# ADR-001: Indoor Positioning Technology for Age Care Facility Room Cleaning Tracker

| Field | Detail |
|---|---|
| **ADR Number** | ADR-001 |
| **Title** | Indoor Positioning Technology for Age Care Facility Room Cleaning Tracker |
| **Status** | Accepted |
| **Date** | 2026-03-24 |
| **Deciders** | Engineering Team, Age Care Facility Operations |
| **Reviewed By** | Infrastructure Lead, Product Owner |

---

## Context

The age care facility requires an automated system to track when a cleaner has spent at least **10 continuous minutes inside a single room**, at which point the room is automatically marked as cleaned. This eliminates manual check-off processes and provides real-time visibility to supervisors via a dashboard integrated with the Property Management System (PMS).

The system must:
- Identify which room a cleaner is currently in
- Track continuous dwell time per room
- Trigger a "room cleaned" event after 10 uninterrupted minutes
- Reset the timer if the cleaner leaves the room before the threshold is met
- Integrate with a Node.js backend and existing age care facility PMS

Three technology options were evaluated for the indoor positioning layer.

---

## Decision Drivers

- **Room-level accuracy** — must reliably distinguish adjacent rooms
- **Infrastructure cost** — hardware cost per room and total deployment cost
- **Battery life** — operational overhead of replacing/recharging tags
- **Integration simplicity** — ease of connecting to Node.js backend
- **Reliability** — resistance to signal interference and environmental factors
- **Regulatory compliance** — no special licensing or approvals required
- **Scalability** — suitable for multi-floor, multi-property deployment

---

## Options Considered

### Option 1 — BLE (Bluetooth Low Energy) Beacons

BLE beacons are small, battery-powered transmitters placed in each room. The cleaner carries a smartphone or BLE tag that detects signal strength (RSSI) from nearby beacons. The strongest signal determines the current room. Dwell time is tracked in the Node.js backend via an MQTT or REST API integration.

**Pros:**
- Low hardware cost (~$10–$30 per room)
- Works with standard smartphones — no dedicated tag needed
- Low setup complexity; widely supported ecosystem
- Easy Node.js integration via MQTT or HTTP
- No regulatory requirements

**Cons:**
- No mesh networking — requires a Wi-Fi gateway in range of every area
- Shorter battery life on active scanning devices (6–18 months)
- Signal bleed through thin walls can cause false room assignments
- Less suitable for large open-plan areas or suites
- No native integration with smart room devices (lighting, thermostats)

---

### Option 2 — Zigbee ✅ SELECTED

Zigbee is a low-power mesh radio protocol operating on 2.4GHz. Each room is fitted with a mains-powered Zigbee router node (which can double as a smart light switch or plug). The cleaner wears a small, long-life Zigbee end-device tag. The tag's signal is picked up by nearby router nodes and relayed through the mesh to a Zigbee coordinator on a Raspberry Pi. Zigbee2MQTT translates device events into MQTT messages that the Node.js backend subscribes to. The backend runs dwell timer logic and publishes room status updates.

**Pros:**
- Self-healing mesh network — devices relay signals hop-by-hop across floors
- Excellent battery life on tags (1–3 years)
- Integrates with existing Zigbee age care facility infrastructure (smart lighting, door locks, thermostats)
- Strong Node.js integration path via Zigbee2MQTT → Mosquitto MQTT broker
- No regulatory requirements
- Supports both local Raspberry Pi deployment and cloud-bridged architecture
- Channel hopping avoids interference with Wi-Fi and BLE

**Cons:**
- Requires a dedicated Zigbee tag for each cleaner (no smartphone app option)
- Slightly higher hardware cost than BLE (~$20–$40 per room node)
- Moderate setup complexity — requires Zigbee2MQTT and MQTT broker configuration
- Room-level accuracy slightly lower than BLE (2–5m vs 1–3m), though sufficient for room detection

---

### Option 3 — GPS (Including Indoor GPS / Pseudolite)

Standard GPS relies on line-of-sight to satellites and does not function reliably indoors. An indoor GPS variant using pseudolites (ground-based transmitters that emit GPS-like signals) was also considered. Pseudolites are used in mining, military, and large warehouse settings to provide GPS-style positioning without satellite dependency.

**Pros:**
- Familiar technology concept
- Standard GPS works well for outdoor campus/resort geofencing
- Pseudolite systems can theoretically achieve room-level accuracy

**Cons:**
- Standard GPS is entirely unsuitable indoors — accuracy degrades to 10–50m+ inside buildings, making room-level detection impossible
- Pseudolite systems cost $100,000–$500,000+ to deploy in a multi-floor age care facility
- Pseudolites require spectrum licensing in Australia (ACMA)
- Standard smartphones cannot receive pseudolite signals — requires specialised hardware
- Extremely high RF engineering complexity to manage multipath reflections
- No viable path to Node.js integration without proprietary SDK
- Not used in any commercial age care facility operations context

---

## Comparison Summary

| Criteria | Option 1: BLE Beacons | Option 2: Zigbee | Option 3: GPS |
|---|---|---|---|
| Room-level accuracy | Good (1–3m) | Good (2–5m) | Not viable indoors |
| Standard smartphone support | Yes | No — dedicated tag | Partial (hybrid only) |
| Tag battery life | 6–18 months | 1–3 years | Hours |
| Mesh networking | No | Yes | No |
| Hardware cost (100 rooms) | $1,500–$3,000 | $2,000–$4,000 | $100,000–$500,000+ |
| Setup complexity | Low | Moderate | Very high |
| Existing age care facility ecosystem fit | Moderate | High | None |
| Regulatory requirements | None | None | Spectrum licence required |
| Indoor reliability | High | High | Very poor |
| Node.js integration | MQTT / REST | MQTT via Zigbee2MQTT | Proprietary SDK only |

---

## Decision

**Option 2 — Zigbee is selected.**

Zigbee is the most operationally suitable technology for age care facility room-level dwell tracking at scale. The mesh networking capability is a decisive advantage in multi-floor age care facility environments, eliminating the need for a gateway in every corridor zone. The 1–3 year tag battery life significantly reduces operational overhead compared to BLE. The integration path through Zigbee2MQTT and Mosquitto MQTT into Node.js is well-established, thoroughly documented, and supports both on-premises and cloud-bridged deployments.

GPS was rejected outright due to fundamental physical limitations indoors and prohibitive cost. BLE remains a viable fallback for single-floor or small age care facility deployments where simplicity is prioritised over mesh resilience.

---

## Architecture Overview

```
Zigbee Tags (cleaners)
        │ radio
        ▼
Zigbee Router Nodes (one per room, mains-powered)
        │ mesh hops
        ▼
Zigbee Coordinator (Raspberry Pi, SONOFF ZBDongle-E)
        │
        ▼
Zigbee2MQTT (running on Raspberry Pi)
        │ MQTT publish
        ▼
Mosquitto Broker (local Pi or cloud VM, port 8883 TLS)
        │ MQTT subscribe
        ▼
Node.js Backend (dwell timer logic, room state machine)
        │
        ▼
Database + Dashboard / PMS Integration
```

**MQTT Topic Structure:**
- `zigbee2mqtt/<device_id>` — device location and signal events (inbound)
- `age care facility/rooms/status` — room cleaned events published by Node.js (outbound)

**Dwell Timer Logic:**
1. Node.js receives MQTT message indicating cleaner tag is associated with a room's router node
2. A 10-minute countdown timer starts for that cleaner/room pair
3. If the tag signal shifts to a different room node, the timer resets
4. A 45-second grace period prevents premature resets from brief corridor crossings
5. On timer completion, a "room cleaned" event is published and persisted to the database

---

## Consequences

**Positive:**
- Reliable, room-accurate indoor positioning with minimal operational maintenance
- Self-healing mesh network tolerates individual node failure without system-wide outage
- Zigbee infrastructure can be shared with smart lighting, thermostat, and door lock systems, reducing total age care facility IoT cost
- Cloud-bridged MQTT architecture supports multi-property management from a single backend

**Negative:**
- Cleaners must carry a dedicated Zigbee tag — a smartphone app is not possible with Zigbee
- Initial setup requires Zigbee2MQTT and MQTT broker configuration, which has a learning curve for teams unfamiliar with IoT middleware
- Tags must be managed, distributed, and occasionally replaced at end of battery life

**Risks and Mitigations:**

| Risk | Likelihood | Mitigation |
|---|---|---|
| Signal bleed between adjacent rooms | Medium | Set RSSI threshold floors; require 3+ consecutive readings before committing room assignment |
| Cleaner forgets to carry tag | Low | Supervisor dashboard shows untracked cleaners in real time |
| Raspberry Pi hardware failure | Low | Run Zigbee2MQTT in Docker with automatic restart; keep a spare Pi on-site |
| MQTT broker connection loss (cloud) | Medium | Local Mosquitto on Pi buffers messages; Node.js reconnects automatically with exponential backoff |

---

## Recommended Hardware (Australia)

| Component | Product | Supplier | Est. Cost |
|---|---|---|---|
| Zigbee coordinator | SONOFF ZBDongle-E | Core Electronics (core-electronics.com.au) | ~$35 |
| Room router nodes | SONOFF ZBMINI or IKEA TRÅDFRI plug | Core Electronics / IKEA | ~$18–$20 each |
| Raspberry Pi host | Raspberry Pi 4 (2GB) | Core Electronics | ~$80 |
| Cleaner tags | Tuya Zigbee personnel tag | AliExpress | ~$20 each |
| Software | Zigbee2MQTT + Mosquitto | Open source — free | $0 |

---
  
## Links and References

- Zigbee2MQTT documentation: https://www.zigbee2mqtt.io
- Mosquitto MQTT broker: https://mosquitto.org
- SONOFF Zigbee devices (AU): https://core-electronics.com.au
- Zigbee specialist supplier (AU): https://shop.dialedin.com.au
- Node.js MQTT client library: https://github.com/mqttjs/MQTT.js

---

*ADR-001 | Status: Accepted | Last updated: 2026-03-24*