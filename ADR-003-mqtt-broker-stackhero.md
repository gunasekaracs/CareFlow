# ADR-003: MQTT Broker Hosting for Hotel Room Cleaning Tracker

| Field | Detail |
|---|---|
| **ADR Number** | ADR-003 |
| **Title** | MQTT Broker Hosting Strategy for Hotel Room Cleaning Tracker |
| **Status** | Accepted |
| **Date** | 2026-03-30 |
| **Deciders** | Engineering Team, Infrastructure Lead |
| **Reviewed By** | Product Owner, Hotel Operations |

---

## Context

The hotel room cleaning tracker system requires an MQTT broker to act as the message bus between the Raspberry Pi BLE scanners on each floor and the ASP.NET Core backend. The Raspberry Pi publishes cleaner location events to MQTT topics in the format `hotel/cleaner/<cleaner_id>/location`, and the .NET backend subscribes to these topics to run dwell timer logic and mark rooms as cleaned.

The broker must be:
- Accessible from both the hotel LAN (Raspberry Pi) and the cloud (ASP.NET Core backend)
- Secured with TLS and credential-based authentication
- Reliable enough for production use across hotel shifts (24/7 operation)
- Maintainable with minimal operational overhead

Two hosting options were evaluated: self-hosting Mosquitto on the team's existing cloud VM, and using Stackhero's managed Mosquitto SaaS.

---

## Decision Drivers

- **Operational overhead** — minimise time spent maintaining broker infrastructure
- **Security** — TLS encryption and device authentication must be in place from day one
- **Reliability** — broker downtime means location events are lost and rooms may not be marked clean
- **Time to production** — faster setup means earlier pilot deployment at the hotel
- **Cost** — must be justifiable for a single-property deployment
- **Scalability** — solution should extend to multiple hotel properties without re-architecture
- **Team capability** — team has strong .NET development skills but limited Linux/DevOps experience

---

## Options Considered

### Option 1 — Self-Hosted Mosquitto on Cloud VM

Install and manage Mosquitto directly on the team's existing cloud server (AWS, Azure, or similar). This involves configuring TLS certificates, setting up password files, managing firewall rules, and handling updates and monitoring manually.

**Pros:**
- No additional monthly cost — runs on the existing cloud VM
- Full control over configuration, including custom ACL rules per device
- Unlimited concurrent connections and message throughput
- No dependency on a third-party SaaS provider
- Well-documented — large community and support resources available

**Cons:**
- Team is responsible for TLS certificate management and renewal
- Security hardening (firewall, fail2ban, port exposure) must be configured manually
- Broker updates and patches are the team's responsibility
- No built-in monitoring dashboard — requires separate tooling (e.g. Grafana, Prometheus)
- Downtime risk if the VM is misconfigured or the team is unavailable
- Scaling to multiple hotel properties requires manual replication of setup per environment
- Requires Linux/DevOps experience to operate confidently in production

---

### Option 2 — Stackhero Managed Mosquitto ✅ SELECTED

Stackhero is a PaaS provider that runs managed Mosquitto instances on dedicated private VMs. Each instance is provisioned in approximately 2 minutes via the Stackhero dashboard, with TLS, authentication, and monitoring included out of the box. The ASP.NET Core backend connects using the standard MQTTnet library — only the host address changes compared to a self-hosted setup.

Stackhero supports both manual user management (up to 20 users) and external API authentication, allowing the .NET backend to dynamically manage cleaner badge credentials via a REST API call rather than maintaining a static password file on the broker.

**Pros:**
- Operational from day one — TLS, security hardening, and monitoring are preconfigured
- Dedicated VM per instance — not shared with other customers, ensuring consistent performance
- Unlimited messages per month — no throttling regardless of message frequency
- Automatic updates, patches, and backups handled by Stackhero
- External API authentication enables dynamic device management from the .NET backend
- Built-in dashboard for connection monitoring and restart/configuration management
- Available as an add-on via Heroku, Azure Marketplace, and direct subscription
- Scales to multiple hotel properties by provisioning additional instances per property
- Zero code change in the .NET backend — only the connection string differs from self-hosted

**Cons:**
- Monthly subscription cost (~$15–$50 USD/month depending on plan)
- Dependency on a third-party provider — if Stackhero has an outage, the broker is unavailable
- Entry plan limited to 200 concurrent connections (sufficient for a single hotel but worth monitoring at scale)
- Less flexibility for advanced custom Mosquitto configurations compared to self-hosting
- Data sovereignty considerations — messages transit Stackhero infrastructure (mitigated by TLS)

---

## Comparison Summary

| Criteria | Option 1: Self-Hosted Mosquitto | Option 2: Stackhero |
|---|---|---|
| Monthly cost | $0 (existing VM) | ~$15–$50 USD/month |
| Setup time | 30–60 minutes | ~2 minutes |
| TLS / security | Manual configuration | Included out of the box |
| Certificate management | Team responsibility | Managed by Stackhero |
| Uptime responsibility | Team | Stackhero |
| Updates and patches | Team | Stackhero |
| Monitoring dashboard | Requires separate tooling | Built in |
| Concurrent connections | Unlimited | 200–1,000 (plan dependent) |
| Messages per month | Unlimited | Unlimited |
| Dynamic device auth | Manual (password file) | External API supported |
| Multi-property scaling | Manual per property | Provision new instance per property |
| DevOps skill required | Moderate | None |
| .NET code change to switch | Host string only | — |

---

## Decision

**Option 2 — Stackhero managed Mosquitto is selected.**

Given the team's primary strength in .NET development rather than Linux infrastructure operations, Stackhero removes the operational risk of a misconfigured or unpatched self-hosted broker going into production. The managed service provides TLS, monitoring, and automatic updates out of the box — capabilities that would require significant additional effort to replicate correctly on a self-hosted instance.

The monthly cost of $15–$50 USD is negligible relative to the development time saved and the operational risk avoided. The entry plan's 200 concurrent connection limit is well above the requirements for a single hotel deployment — a typical hotel with 10 cleaners and a handful of Raspberry Pi scanners will use fewer than 20 concurrent connections at peak.

The Stackhero approach also provides a clear multi-property scaling path: each hotel property receives its own Stackhero Mosquitto instance, and the .NET backend is configured with the relevant connection string per property. No infrastructure changes are needed beyond provisioning a new instance.

Self-hosted Mosquitto remains the recommended fallback if the team's operational capability grows, or if cost becomes a concern at larger scale.

---

## Architecture Overview

```
Raspberry Pi (each floor)
        │
        │  MQTT publish over TLS (port 8883)
        │  topic: hotel/cleaner/<id>/location
        ▼
Stackhero Mosquitto Instance
(dedicated VM, TLS included, managed updates)
        │
        │  MQTTnet subscribe (TLS, credentials)
        ▼
ASP.NET Core background worker
(DwellTimerService — 10-min timer logic)
        │
        ▼
SQL Server / PostgreSQL
(room status + clean history)
        │
        ▼
Dashboard + PMS integration
```

**MQTT topic structure:**
- `hotel/cleaner/<cleaner_id>/location` — Raspberry Pi publishes badge detection events
- `hotel/rooms/status` — ASP.NET Core publishes room cleaned events (optional, for dashboard)

**Authentication approach:**
- Raspberry Pi scanners authenticated via Stackhero manual user management (static credentials per Pi)
- Cleaner badge credentials managed dynamically via Stackhero external API authentication, called from the .NET backend when a new cleaner is onboarded

---

## .NET Connection Configuration

The only difference between self-hosted and Stackhero in the .NET codebase is the broker host:

```csharp
// appsettings.json
{
  "Mqtt": {
    "Host": "xxxxxxxx.stackhero-network.com",
    "Port": 8883,
    "Username": "dotnetbackend",
    "Password": "<from environment variable>"
  }
}

// MqttWorker.cs
var options = new MqttClientOptionsBuilder()
    .WithTcpServer(config["Mqtt:Host"], int.Parse(config["Mqtt:Port"]))
    .WithCredentials(config["Mqtt:Username"],
        Environment.GetEnvironmentVariable("MQTT_PASSWORD"))
    .WithTlsOptions(o => o.UseTls())
    .Build();
```

Migrating back to self-hosted Mosquitto in the future requires only updating `Mqtt:Host` in `appsettings.json` — no code changes.

---

## Consequences

**Positive:**
- Production-ready broker available within minutes of account creation
- TLS and security hardening require no team effort
- Automatic updates eliminate broker vulnerability exposure
- Built-in monitoring dashboard provides visibility into connection count and message rates
- Dynamic device authentication via external API allows clean onboarding of new cleaner badges from the .NET admin interface
- Clear upgrade path to higher connection plans if the hotel grows or multi-property deployment begins

**Negative:**
- Ongoing monthly cost ($15–$50 USD) that did not exist with the self-hosted option
- Third-party dependency introduces an external failure point — broker availability is outside the team's direct control
- If Stackhero discontinues the service, migration to self-hosted or another provider is required (mitigated by the fact that the connection string is the only configuration change)

**Risks and Mitigations:**

| Risk | Likelihood | Mitigation |
|---|---|---|
| Stackhero outage causes broker downtime | Low | Raspberry Pi queues messages locally and retries on reconnect; MQTTnet handles reconnection automatically |
| Stackhero discontinues service | Very low | Connection string is the only coupling — migration to self-hosted Mosquitto or HiveMQ Cloud takes under an hour |
| 200 connection limit reached | Low | Upgrade to next Stackhero plan; single hotel with 10 cleaners and 5 Raspberry Pis uses ~15 connections at most |
| Message interception in transit | Very low | All connections use TLS (port 8883) — plaintext connections are disabled by default on Stackhero |
| Cleaner badge credentials leaked | Low | Credentials managed via Stackhero external API — no static password file on the broker |

---

## Links and References

- Stackhero Mosquitto service: https://www.stackhero.io/en/services/Mosquitto/benefits
- Stackhero Mosquitto documentation: https://www.stackhero.io/en/services/Mosquitto/documentations
- Stackhero Mosquitto pricing: https://www.stackhero.io/en/services/Mosquitto/pricing
- Stackhero external API authentication example: https://github.com/stackhero-io/mosquittoGettingStarted
- MQTTnet .NET library: https://github.com/dotnet/MQTTnet
- Mosquitto self-hosted documentation: https://mosquitto.org/documentation/

---

*ADR-003 | Status: Accepted | Last updated: 2026-03-30*
