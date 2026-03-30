# ADR-004: Cleaner Wearable Tag for Hotel Room Cleaning Tracker

| Field | Detail |
|---|---|
| **ADR Number** | ADR-004 |
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

---

## Options Considered

### Option 1 — ID Badge / Card Beacon ✅ SELECTED

A credit-card sized BLE beacon worn on the cleaner's existing staff ID lanyard. The Minew C10 Card Beacon is the recommended product — it uses Bluetooth 5.0, broadcasts iBeacon format, and is designed specifically for personnel management and indoor positioning. It looks identical to a standard staff ID card and can replace or sit alongside the cleaner's existing hotel ID.

**Recommended product:** Minew C10 Card Beacon
- Bluetooth 5.0, 100m broadcast range
- iBeacon and Eddystone format support
- Built-in accelerometer (detects movement/stillness)
- Hidden panic button for emergency alerting (future use)
- Replaceable CR2032 battery — up to 12 months battery life at standard broadcast interval
- Dimensions: standard credit card size
- Configurable via free FeasyBeacon iOS/Android app

**Pros:**
- Cleaners already wear a staff ID lanyard — no new habit required
- Indistinguishable from a regular ID card — professional appearance
- Passive broadcast — zero interaction from cleaner during shift
- Up to 12-month battery life — annual replacement cycle only
- Built-in accelerometer enables motion detection — can distinguish active cleaning from idle
- Panic button provides future value for staff safety alerting
- Standard iBeacon format compatible with the Raspberry Pi `bluepy` scanner out of the box
- Low cost — ~$20–$25 AUD per tag
- Available from AliExpress and Minew directly — no specialist local supplier needed
- Configurable UUID, Major, and Minor values per tag for unique cleaner identification

**Cons:**
- Card can be left at the locker if cleaner forgets to attach it to their lanyard
- Shared lanyards between shifts could cause identity confusion if tags are not cleaner-specific
- CR2032 battery replacement requires a small flathead screwdriver — not user-serviceable by most cleaners

---

### Option 2 — Wristband Beacon

A BLE wristband worn on the cleaner's wrist, similar to a fitness tracker. The BeaconTrax Trax10195 Wristband BLE Beacon is the reference product — it is BLE 5.0 compatible, has a built-in accelerometer, and is designed for people flow management and personnel tracking.

**Reference product:** BeaconTrax Trax10195
- BLE 5.0
- Built-in accelerometer for movement, vibration, and fall detection
- Configurable over-the-air
- Replaceable battery

**Pros:**
- Always on the cleaner's body — cannot be left behind at the locker
- Accelerometer data provides richer motion context
- Fall detection capability is valuable for lone worker safety
- Waterproof variants available — suitable for cleaning work involving water

**Cons:**
- Some cleaners may find wristbands uncomfortable during physical cleaning work, especially with rubber gloves
- Cultural or religious objections to wearing wristbands are possible in a diverse hotel workforce
- Higher cost than card beacons — ~$40–$80 USD per unit depending on supplier
- Requires charging or battery replacement more frequently than a card beacon
- Feels more invasive from a staff perspective — some cleaners may object to wearing a tracking device on their body
- Not available on Amazon AU — must order internationally from BeaconTrax with longer lead times

---

### Option 3 — Key Fob Beacon

A compact BLE key fob that the cleaner clips to their belt loop, cleaning trolley, or uniform pocket. The GAO RFID Wearable Key Chain Style Beacon is the reference product.

**Reference product:** GAO RFID SKU#127048 Key Chain Beacon
- iBeacon and Eddystone format
- Compact key fob form factor
- Replaceable battery

**Pros:**
- Familiar form factor — similar to a door access fob most hotel staff already use
- Can be clipped to cleaning trolley as an alternative to body wear
- Low cost — comparable to card beacons
- No wristband comfort concerns

**Cons:**
- If attached to the trolley rather than the cleaner, it tracks the trolley not the person — unreliable for room presence detection if cleaner leaves the trolley in the corridor
- Easy to lose or drop during cleaning work
- Less professional appearance than a card badge in a hotel context
- Not purpose-built for personnel tracking — primarily an asset tag
- Limited accelerometer and sensor options compared to card or wristband alternatives
- Supplier (GAO RFID) ships from North America — longer delivery to Australia

---

## Comparison Summary

| Criteria | Option 1: ID Badge (Minew C10) | Option 2: Wristband (Trax10195) | Option 3: Key Fob (GAO RFID) |
|---|---|---|---|
| Form factor | Credit card on lanyard | Wrist-worn band | Belt clip / key chain |
| Worn by cleaner | Yes — on existing lanyard | Yes — on wrist | Sometimes — may go on trolley |
| Tracks person reliably | Yes | Yes | Only if worn, not on trolley |
| Battery life | ~12 months (CR2032) | Shorter (rechargeable) | ~12 months (coin cell) |
| Bluetooth version | BLE 5.0 | BLE 5.0 | BLE 4.x |
| Accelerometer | Yes | Yes (+ fall detection) | Limited |
| Panic button | Yes | Optional | No |
| Professional appearance | High — looks like staff ID | Medium | Low |
| Staff acceptance risk | Low | Medium (comfort concerns) | Medium (may use on trolley) |
| Cost per unit (AUD) | ~$20–$25 | ~$60–$110 | ~$25–$40 |
| Australian availability | AliExpress / minew.com | BeaconTrax (international) | GAO RFID (international) |
| iBeacon compatible | Yes | Yes | Yes |
| Raspberry Pi scanner compatible | Yes | Yes | Yes |

---

## Decision

**Option 1 — Minew C10 ID Badge / Card Beacon is selected.**

The ID badge form factor is the clear choice for a hotel cleaning environment. Cleaners already wear a staff ID on a lanyard as part of their uniform — replacing or supplementing it with the Minew C10 requires no change in behaviour whatsoever. The tag broadcasts passively, requires no charging, and lasts up to 12 months on a single CR2032 battery. Its professional appearance is consistent with hotel staff presentation standards, and its built-in accelerometer and panic button provide future value for motion-aware tracking and lone worker safety.

The wristband option, while technically capable, introduces comfort and cultural concerns that are inappropriate to impose on a diverse hotel cleaning workforce without explicit consultation. The key fob option introduces tracking unreliability if cleaners attach it to their trolley rather than their person.

At ~$20–$25 AUD per tag, the Minew C10 is the most cost-effective option that meets all requirements. A team of 10 cleaners can be equipped for ~$200–$250 AUD — a negligible line item in the overall system cost.

---

## Architecture Context

This decision sits at the hardware layer of the system. The Minew C10 tag is the signal source detected by the Raspberry Pi scanner (ADR-004), which publishes to the Stackhero MQTT broker (ADR-003), which is consumed by the ASP.NET Core backend running dwell timer logic.

```
Minew C10 badge (cleaner wears on lanyard)
        │  BLE 5.0 iBeacon broadcast (UUID = cleaner ID)
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
Each Minew C10 tag is configured with a unique UUID (or unique Major/Minor combination) that maps to a specific cleaner ID in the system. This mapping is stored in the Raspberry Pi scanner configuration and in the .NET backend database. Configuration is performed once during onboarding using the free FeasyBeacon mobile app.

```python
# Raspberry Pi scanner — badge to cleaner mapping
BADGE_MAP = {
    "UUID-cleaner-001": "cleaner_001",  # Jane Smith
    "UUID-cleaner-002": "cleaner_002",  # John Doe
    # Add new cleaners here on onboarding
}
```

---

## Operational Procedures

**Onboarding a new cleaner:**
1. Take a new Minew C10 tag out of stock
2. Open the FeasyBeacon app on a supervisor's phone
3. Configure the tag UUID to match the cleaner's ID in the system
4. Add the UUID → cleaner ID mapping to the Pi scanner config
5. Clip the tag to the cleaner's lanyard alongside their hotel ID card

**Battery replacement:**
- CR2032 batteries last approximately 12 months at default broadcast settings
- Supervisor checks battery health monthly via the Raspberry Pi scanner logs (tags with weak RSSI may indicate low battery)
- Replacement batteries available from any supermarket or electronics store — Coles, Woolworths, Jaycar

**Lost or damaged tag:**
- Issue a replacement Minew C10 from stock
- Configure the new tag with the same UUID as the lost tag
- No backend changes required

---

## Consequences

**Positive:**
- Zero change to cleaner workflow — tag is worn on the existing lanyard automatically
- Passive broadcast requires no interaction, charging, or app from the cleaner
- 12-month battery cycle means one annual maintenance round for the entire tag fleet
- Panic button and accelerometer provide immediate future value for staff safety features
- Professional card form factor is consistent with hotel uniform standards

**Negative:**
- Tag must be physically onboarded per cleaner using the FeasyBeacon app — adds a small setup step per new staff member
- If a cleaner forgets their lanyard, location tracking is lost for that cleaner's shift
- CR2032 replacement requires a small tool — not easily self-serviced by cleaners; supervisor must handle

**Risks and Mitigations:**

| Risk | Likelihood | Mitigation |
|---|---|---|
| Cleaner leaves badge at locker | Low | Supervisor dashboard shows cleaners with no recent signal; supervisor checks in at shift start |
| Tag battery dies mid-shift | Low | Scanner logs report weakening RSSI before complete failure; monthly battery health checks |
| Two cleaners swap lanyards | Very low | Each tag UUID is unique per cleaner; swap would mis-attribute cleaning credit — addressed in onboarding training |
| Tag damaged by cleaning chemicals | Low | Minew C10 has a sealed plastic casing; keep away from direct chemical spray |
| FeasyBeacon app unavailable for configuration | Very low | UUID can also be set via the Minew management SDK; configuration only needed at onboarding |

---

## Recommended Stock Levels

For a 100-room hotel with 10 active cleaners:

| Item | Quantity | Reason |
|---|---|---|
| Minew C10 tags (active) | 10 | One per cleaner |
| Minew C10 tags (spare) | 3 | Replacement for lost or damaged tags |
| CR2032 batteries (spare) | 20 | Annual replacement stock for 12+ months |

**Total tag investment: ~$325 AUD** (13 tags × ~$25 AUD)

---

## Links and References

- Minew C10 Card Beacon product page: https://www.minew.com/product/c10-card-beacon/
- Minew C10 on AliExpress: Search "Minew C10 card beacon"
- BeaconTrax Trax10195 Wristband Beacon: https://www.beacontrax.com/product/trax10195-wearable-wristband-ble-beacon/
- GAO RFID Key Chain Beacon: https://gaorfid.com/devices/ble/ble-beacons/
- FeasyBeacon configuration app: Available on iOS App Store and Google Play
- ADR-002: Indoor positioning technology decision (BLE Beacons selected)
- ADR-003: MQTT broker decision (Stackhero selected)
- ADR-004: BLE scanning approach decision (Raspberry Pi selected)

---

*ADR-004 | Status: Accepted | Last updated: 2026-03-30*
