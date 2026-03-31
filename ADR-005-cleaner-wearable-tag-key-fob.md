# ADR-005: Cleaner Wearable Tag for Hotel Room Cleaning Tracker

| Field | Detail |
|---|---|
| **ADR Number** | ADR-005 |
| **Title** | Cleaner Wearable BLE Tag Selection for Hotel Room Cleaning Tracker |
| **Status** | Accepted |
| **Date** | 2026-03-30 |
| **Deciders** | Engineering Team, Hotel Operations |
| **Reviewed By** | Infrastructure Lead, Product Owner |

---

## Context

The hotel room cleaning tracker system requires each cleaner to carry or wear a BLE transmitter that continuously broadcasts their identity. The Raspberry Pi scanner on each floor (per ADR-004) detects this signal and determines which room the cleaner is in based on RSSI proximity to fixed room reference beacons.

Rather than requiring cleaners to carry a smartphone or dedicated handheld device, the preferred approach is a passive wearable tag that broadcasts automatically without any interaction from the cleaner. The tag must be comfortable, practical for cleaning work, and require minimal maintenance.

Three wearable form factors were evaluated: an ID badge/card, a wristband, and a key fob.

---

## Decision Drivers

- **Comfort and practicality** — cleaners perform physical work; the tag must not impede movement
- **Zero interaction required** — the tag must broadcast passively without the cleaner needing to do anything
- **Familiarity** — the form factor should feel natural in a hotel staff context
- **Battery life** — minimise how frequently tags need to be recharged or have batteries replaced
- **Loss and damage risk** — hotel cleaning is physically demanding; the tag must survive the work environment
- **Cost** — per-tag cost must be justifiable for a full cleaning team deployment
- **Integration with existing hotel processes** — ideally fits into something staff already wear or carry
- **BLE compatibility** — must broadcast standard iBeacon or Eddystone format compatible with the Raspberry Pi scanner
- **Flexibility of attachment** — option to attach to person or cleaning trolley depending on workflow

---

## Options Considered

### Option 1 — ID Badge / Card Beacon

A credit-card sized BLE beacon worn on the cleaner's existing staff ID lanyard. The Minew C10 Card Beacon is the reference product — it uses Bluetooth 5.0, broadcasts iBeacon format, and is designed specifically for personnel management and indoor positioning. It looks identical to a standard staff ID card.

**Reference product:** Minew C10 Card Beacon
- Bluetooth 5.0, 100m broadcast range
- iBeacon and Eddystone format support
- Built-in accelerometer and hidden panic button
- Up to 12 months battery life (CR2032)
- Standard credit card dimensions

**Pros:**
- Fits naturally on the existing staff ID lanyard — no new habit required
- Professional appearance consistent with hotel uniform standards
- Passive broadcast — zero interaction from cleaner
- Up to 12 months battery life
- Built-in panic button provides future staff safety value
- Low cost (~$20–$25 AUD per tag)

**Cons:**
- If the cleaner forgets their lanyard, tracking is lost for that shift
- Shared lanyards between shift workers could cause identity confusion
- CR2032 battery replacement requires a small tool — not easily self-serviced
- Card can be misplaced if removed from lanyard for any reason

---

### Option 2 — Wristband Beacon

A BLE wristband worn on the cleaner's wrist. The BeaconTrax Trax10195 Wristband BLE Beacon is the reference product — BLE 5.0, built-in accelerometer with fall detection, designed for people flow management and personnel tracking.

**Reference product:** BeaconTrax Trax10195
- BLE 5.0
- Accelerometer with fall detection
- Configurable over-the-air
- Waterproof variants available

**Pros:**
- Always on the cleaner's body — cannot be left behind
- Fall detection capability for lone worker safety
- Waterproof variants suitable for cleaning work

**Cons:**
- Wristbands may be uncomfortable during physical cleaning work, especially with rubber gloves
- Cultural or religious objections to wristbands possible in a diverse hotel workforce
- Higher cost (~$60–$110 AUD per unit)
- More invasive from a staff perspective
- Not available on Amazon AU — international order required with longer lead times

---

### Option 3 — Key Fob Beacon ✅ SELECTED

A compact BLE key fob that the cleaner clips to their belt loop, uniform pocket, or cleaning trolley handle. The GAO RFID Wearable Key Chain Style Beacon is the reference product — it combines iBeacon and Eddystone formats in a small, rugged form factor familiar to hotel staff who already use key fobs for door access.

**Recommended product:** GAO RFID SKU#127048 Key Chain Beacon
- iBeacon and Eddystone (UID, URL, TLM) format support
- Compact key fob form factor with clip attachment
- Replaceable coin cell battery — up to 12 months at standard broadcast interval
- 2.4GHz BLE broadcast
- Compatible with standard Raspberry Pi `bluepy` BLE scanner
- Configurable UUID, TX power, and broadcast interval

**Pros:**
- Extremely familiar form factor — hotel staff already use key fobs for room access and staff areas
- Clips directly to belt loop, uniform, lanyard, or trolley handle — flexible attachment options
- Passive broadcast — zero interaction from cleaner during shift
- Compact and lightweight — does not impede cleaning movement
- Rugged plastic casing suitable for physical cleaning environment
- Up to 12 months coin cell battery life — low maintenance overhead
- Low cost — comparable to card beacons (~$25–$40 AUD per unit)
- Standard iBeacon format compatible with Raspberry Pi `bluepy` scanner out of the box
- Unique UUID per fob for individual cleaner identification
- Can be attached to cleaning trolley as a deliberate asset-level tracking option if required

**Cons:**
- If clipped to cleaning trolley rather than the cleaner's person, it tracks the trolley not the individual — must be attached to uniform or belt loop for accurate person tracking
- Smaller form factor increases risk of being dropped or lost during work
- Less professional appearance compared to an ID card badge in a premium hotel context
- No built-in accelerometer or panic button on the base model
- Supplier (GAO RFID) ships from North America — 2–3 week delivery to Australia; AliExpress alternatives available faster

---

## Comparison Summary

| Criteria | Option 1: ID Badge (Minew C10) | Option 2: Wristband (Trax10195) | Option 3: Key Fob (GAO RFID) |
|---|---|---|---|
| Form factor | Credit card on lanyard | Wrist-worn band | Belt clip / key chain |
| Worn by cleaner | Yes — on existing lanyard | Yes — on wrist | Yes — clipped to belt or uniform |
| Tracks person reliably | Yes | Yes | Yes (if worn on person, not trolley) |
| Battery life | ~12 months (CR2032) | Shorter (rechargeable) | ~12 months (coin cell) |
| Bluetooth version | BLE 5.0 | BLE 5.0 | BLE 4.x–5.0 |
| Accelerometer | Yes | Yes (+ fall detection) | No (base model) |
| Panic button | Yes | Optional | No |
| Staff familiarity | Medium (lanyard ID card) | Low (wristband) | High (key fob) |
| Comfort during cleaning | High | Medium (gloves issue) | High |
| Staff acceptance risk | Low | Medium | Low |
| Flexible attachment | No — lanyard only | No — wrist only | Yes — belt, pocket, trolley |
| Cost per unit (AUD) | ~$20–$25 | ~$60–$110 | ~$25–$40 |
| Australian availability | AliExpress / minew.com | BeaconTrax (international) | GAO RFID / AliExpress |
| iBeacon compatible | Yes | Yes | Yes |
| Raspberry Pi scanner compatible | Yes | Yes | Yes |

---

## Decision

**Option 3 — Key Fob Beacon (GAO RFID SKU#127048) is selected.**

The key fob form factor is the most practical and universally accepted choice for a hotel cleaning environment. Hotel cleaning staff are already accustomed to carrying key fobs for room access, staff area entry, and equipment lockers — adding a BLE fob to an existing key ring or belt clip requires no change in behaviour and no new habit to form.

The key fob is lightweight, rugged, and unobtrusive during physical cleaning work. Unlike a wristband, it does not interfere with rubber gloves and raises no cultural or comfort concerns across a diverse workforce. Unlike an ID card badge, it does not depend on the cleaner wearing a lanyard and is less likely to be removed during work.

The flexible attachment option is a genuine operational advantage — in future, if the hotel chooses to track cleaning trolleys as assets in addition to individual cleaners, the same fob type can serve both purposes without hardware changes.

At ~$25–$40 AUD per unit, the cost is slightly higher than the Minew C10 card badge but significantly lower than the wristband option. For a team of 10 cleaners, the total tag investment remains under $400 AUD.

The key consideration — attaching the fob to the person rather than the trolley — must be addressed in onboarding training and enforced through standard operating procedure.

---

## Architecture Context

This decision sits at the hardware layer of the system. The key fob tag is the signal source detected by the Raspberry Pi scanner (ADR-004), which publishes to the Stackhero MQTT broker (ADR-003), consumed by the ASP.NET Core backend running dwell timer logic.

```
GAO RFID key fob (clipped to cleaner's belt loop or uniform)
        │  BLE iBeacon broadcast (UUID = cleaner ID)
        ▼
Feasycom DA14531 beacons (fixed, one per room)
        │  Raspberry Pi detects both signals, calculates closest room
        ▼
Raspberry Pi 4 scanner (ADR-004)
        │  MQTT publish → Stackhero (ADR-003)
        ▼
ASP.NET Core DwellTimerService
        │  10-min timer → room marked clean
        ▼
Dashboard + PMS
```

**Tag configuration per cleaner:**
Each key fob is configured with a unique UUID that maps to a specific cleaner ID in the system. This mapping is stored in the Raspberry Pi scanner configuration and in the .NET backend database. Configuration is performed once during onboarding using a BLE configuration app.

```python
# Raspberry Pi scanner — fob UUID to cleaner mapping
BADGE_MAP = {
    "UUID-cleaner-001": "cleaner_001",  # Jane Smith
    "UUID-cleaner-002": "cleaner_002",  # John Doe
    # Add new cleaners here on onboarding
}
```

---

## Operational Procedures

**Onboarding a new cleaner:**
1. Take a new GAO RFID key fob from stock
2. Configure the fob UUID to match the cleaner's ID using a BLE configuration app
3. Add the UUID → cleaner ID mapping to the Pi scanner config and backend database
4. Issue the fob to the cleaner — instruct them to clip it to their belt loop or uniform pocket (not to the trolley)
5. Verify the fob appears in the Raspberry Pi scanner logs during a test walk-through

**Staff instruction (mandatory at onboarding):**
- The fob must be clipped to your belt loop, uniform pocket, or lanyard
- Do not clip the fob to the cleaning trolley — it must stay on your person
- If you lose the fob, report it to your supervisor immediately for a replacement

**Battery replacement:**
- Coin cell batteries last approximately 12 months at standard broadcast settings
- Supervisor checks battery health monthly via the Pi scanner logs — tags with consistently weak RSSI may indicate low battery
- Replacement coin cell batteries available from Coles, Woolworths, or Jaycar

**Lost or damaged fob:**
- Issue a replacement fob from stock
- Configure the new fob with the same UUID as the lost fob using the BLE config app
- No backend database changes required

---

## Consequences

**Positive:**
- Universally familiar form factor — no training required beyond attachment instruction
- No comfort, cultural, or BYOD concerns across a diverse cleaning workforce
- Flexible attachment means the same hardware can serve both personal tracking and trolley asset tracking in future
- Rugged construction appropriate for physical cleaning environment
- 12-month battery cycle means one annual maintenance round for the entire fob fleet
- Low cost — entire fleet of 10 cleaners equipped for under $400 AUD

**Negative:**
- Must be worn on person, not trolley — requires clear onboarding instruction and occasional supervisor enforcement
- Smaller form factor increases drop and loss risk compared to a card badge
- No built-in accelerometer or panic button on base model — limits future motion-aware or safety features without upgrading to a higher-spec fob
- GAO RFID ships from North America — order lead time of 2–3 weeks; plan ahead for initial stock and replacements

**Risks and Mitigations:**

| Risk | Likelihood | Mitigation |
|---|---|---|
| Cleaner clips fob to trolley instead of person | Medium | Address in onboarding training and SOP; supervisor spot-checks during first week |
| Fob lost or dropped during cleaning | Medium | Keep 3 spare fobs in stock; supervisor dashboard alerts when a cleaner has no recent signal |
| Battery dies mid-shift | Low | Pi scanner logs report weakening RSSI before failure; monthly battery health checks |
| Fob UUID conflict between two cleaners | Very low | UUID assignment managed centrally in the backend database during onboarding |
| GAO RFID stock delay from North America | Low | Order via AliExpress (search "BLE key fob iBeacon") for 1–2 week delivery; same BLE format |
| Fob signal interference from metal belt buckles | Low | Test during pilot — reposition clip slightly if RSSI is consistently weak |

---

## Recommended Stock Levels

For a 100-room hotel with 10 active cleaners:

| Item | Quantity | Reason |
|---|---|---|
| GAO RFID key fob tags (active) | 10 | One per cleaner |
| GAO RFID key fob tags (spare) | 3 | Replacement for lost or damaged fobs |
| Replacement coin cell batteries | 20 | Annual replacement stock for 12+ months |

**Total tag investment: ~$365–$525 AUD** (13 fobs × ~$28–$40 AUD)

---

## Links and References

- GAO RFID Key Chain Style Beacon: https://gaorfid.com/devices/ble/ble-beacons/
- AliExpress BLE key fob alternatives: Search "BLE iBeacon key fob" on aliexpress.com
- Minew C10 Card Beacon (Option 1 reference): https://www.minew.com/product/c10-card-beacon/
- BeaconTrax Trax10195 Wristband Beacon (Option 2 reference): https://www.beacontrax.com/product/trax10195-wearable-wristband-ble-beacon/
- ADR-002: Indoor positioning technology decision (BLE Beacons selected)
- ADR-003: MQTT broker decision (Stackhero selected)
- ADR-004: BLE scanning approach decision (Raspberry Pi selected)

---

*ADR-005 | Status: Accepted | Last updated: 2026-03-30*
