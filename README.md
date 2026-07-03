# RemoteGenius

> RemoteGenius (RG Platform) is a fleet-management platform for remote IP-enabled devices — anything with a web management dashboard or an internal API, from network appliances and industrial controllers to cameras, encoders, and sensors — that reaches hardware behind CGNAT and strict firewalls using outbound-only reverse tunnels, unifies mixed-vendor fleets under one device-shadow schema, and runs on a regionally distributed, air-gap-capable cluster architecture.

**RemoteGenius** (the **RG Platform**) manages any IP-enabled device that exposes a **web GUI dashboard** and/or an **internal API** — controllers, switches, routers, PDUs, UPS units, access points, NVRs, KVMs, environmental sensors, media gateways, cameras, encoders, industrial PLCs, and anything else with an HTTP management interface. It reaches those devices when they sit on cellular links, behind carrier-grade NAT (CGNAT), on customer LANs, or on sites the operator doesn't administer, and presents them through one dashboard, one credential, and one permission model across every device, vendor, site, and region.

The two things RG hooks into are the two things almost every modern device already ships with: its **web management interface** (tunneled through the platform and gated by RG's access control, so operators reach it without a VPN, bastion, or port forwarding) and its **internal API** (normalized by an adapter into a canonical device shadow). If a device has either, RG can manage it. If it has neither, a small **bridge appliance** represents it and it joins the fleet anyway.

This organization hosts the public, sanitized specifications: the unified device data model, connectivity model, adapter contract, and reference clients. Start with **[rg-specifications](https://github.com/remotegenius/rg-specifications)** for schemas and runnable examples, or the docs at **[docs.remotegeni.us](https://docs.remotegeni.us)**.

---

## What RG solves

Four hard problems every operator of remote hardware eventually hits:

- **Reaching devices on networks you don't own.** Cellular, CGNAT, hostile corporate firewalls, moving vehicles, customer premises with no admin access. RG devices initiate every connection outbound, so there are no public IPs, no inbound firewall rules, and no port forwarding on the customer side.
- **Managing a mixed-vendor fleet without a bespoke integration per SKU.** Every device — regardless of vendor, model, or firmware — is expressed through one canonical *device shadow*; vendor-specific translation happens inside the platform via adapters, driven by each device's own API. Where an operation isn't modeled, the device's native web GUI is tunneled through instead.
- **Staying operational when the cloud is down or the uplink is flaky.** Clusters run autonomously; nothing on the device data path depends on the global control plane being reachable. Air-gapped sites are a first-class deployment mode, not a stripped-down subset.
- **Doing all of the above without giving up tenant isolation, compliance, or auditability.** Two-tier RBAC, per-component identity, and audit-at-every-layer are invariants of the design, not toggles.

---

## Architecture in one screen

RG is three cooperating layers, each with a clear responsibility and a clear failure boundary, so a disruption in one layer does not cascade into the others.

| Layer | Owns | Failure behavior |
|---|---|---|
| **Cloud — global control plane** | Accounts, orgs, role/permission catalog, cluster registry, billing, long-retention audit, short-lived cluster-scoped token issuance | Bounded by design: holds no device credentials, sits on no device data path. If it's down, operators lose the global dashboard, not their fleet. |
| **Cluster — regional execution plane** | Device identity, pairing, secure channels, config, adapters, telemetry, local audit, local RBAC | Multiple interchangeable nodes over an encrypted private mesh with a consensus-backed datastore. Loss of a node, a region, or a whole provider does not stop the cluster. |
| **Device — edge agents & bridges** | Outbound secure channel, liveness/config/firmware reporting, atomic config application, signed self-update | Reconnects with exponential backoff; re-allocates endpoints; degrades gracefully across intermittent connectivity. |

Users authenticate against the cloud, then exchange their session for a short-lived, cluster-scoped credential to operate a specific cluster. Devices authenticate against their own cluster only, using credentials issued during pairing — they never hold cloud credentials and never talk to the cloud directly.

---

## Device connectivity model

The reason RG works where generic cloud-IoT offerings don't.

**Outbound-only, zero inbound.** Every device-to-platform connection is initiated by the device, outbound, over an encrypted channel. No public IP, no inbound firewall rule, no port forwarding, no UPnP. This eliminates the largest attack surface in traditional remote device management — no managed device or bridge appliance ever accepts an inbound connection from the internet.

**Persistent reverse channels expose the device's local services.** Once paired, a device holds a long-lived encrypted reverse channel to its cluster node. Inside that single channel, any of the device's local services are mapped to allocated ports on the cluster side (`DevicePortAllocation`): its **web GUI** (typically HTTP on `127.0.0.1`), its **internal API**, its telemetry stream, and — where the device serves it — a media stream. The cluster's adapters and user-facing APIs reach into the device through these mapped endpoints. Channel health is supervised continuously; stale endpoints are reclaimed automatically.

**Pull-based control.** The agent polls its cluster on a short cadence for control signals (new config, reboot, reprovision, scheduled upgrade). Pull survives intermittent connectivity and never requires the cluster to "find" the device.

**Pairing with no pre-shared secret.** A short-lived, operator-visible code plus a fresh asymmetric key generated on the device. The code is valid for minutes and useless without explicit operator approval; the asymmetric key never leaves the device and is the permanent trust anchor. Per-serial lockouts and rate limits stop brute-force and enumeration.

---

## The data model that makes RG different

RG treats the **managed device** (the thing the operator cares about — a controller, a camera, a switch, a sensor) and the **bridge** (the component carrying its traffic to the cluster) as two separate entities with separate lifecycles. A device is *attached to* a bridge; a bridge is not part of a device.

A bridge takes one of two **form factors**, first-class in the model rather than implicit in deployment:

- **Agent form factor — embedded on the device.** The bridge agent runs on the managed device itself and tunnels the device's native management interface (typically HTTP on `127.0.0.1`) out to its cluster node. Reference example: an embedded device where the agent installs under a BusyBox init as root and tunnels port 80 back to the cluster.
- **Appliance form factor — standalone bridge.** The agent runs on a small RG-supplied appliance that fronts one or more external devices over Ethernet and tunnels each device's local services out. Reference example: a compact Linux appliance (e.g. NanoPi R3S LTS) running the agent as a `systemd` unit under a non-root user with `sudo`.

Why the separation is worth enforcing in the schema:

- **Device turnover without re-pairing.** An appliance bridge is paired and hardened once, then trusted as a long-lived cluster citizen. Swap, RMA, or replace the device it fronts without touching the cluster's trust store.
- **Multiple devices per bridge.** Devices reference their bridge by nullable foreign key, not the reverse, so one appliance can front several external devices.
- **Clean capability boundary.** An agent-form-factor bridge inherits firmware/model from the device it lives on, so those fields belong on the device row. An appliance has its own firmware/model, independent of any attached device. A DB constraint enforces that `form_factor = 'agent'` implies no bridge-level firmware/model.
- **Attachment as observable state.** Devices carry an explicit `attachment_state` — `never_attached`, `attached`, `detached` — so the cluster distinguishes "bridge reachable, device unplugged" from "bridge unreachable" from "device never paired," without heuristics.

Model shape (see full JSON Schemas in [rg-specifications/schemas](https://github.com/remotegenius/rg-specifications/tree/main/schemas)):

- `bridge.form_factor` — `CHECK (form_factor IN ('agent','appliance'))`; agent bridges legally cannot carry device-level firmware/model.
- `device.bridge_id` — nullable FK to `bridges(id)` with `ON DELETE SET NULL`; an optional link, not an identity.
- `device.attachment_state` — `CHECK (... IN ('attached','detached','never_attached'))`; `attached` implies `bridge_id IS NOT NULL`.

---

## Unified device control — the device shadow

Every managed device is modeled as a **device shadow**: a canonical, vendor-neutral representation of current settings, reported telemetry, supported actions, firmware state, and change history. This is what lets a fleet of unrelated hardware — different vendors, models, and firmware generations — present as one uniform control surface.

Behind the shadow sit per-vendor, per-model, per-firmware **adapters**. An adapter speaks to the device's **internal API** and translates canonical settings into the vendor's native wire format and back. It normalizes types across vendors, expresses invalidation chains (toggling DHCP refreshes IP-related fields so the UI never shows stale data), expresses conditional visibility (hide static-IP inputs when DHCP is on), reports validation errors in operator terms, and manages firmware uploads with checksums and rollback awareness.

Adapters are selected per device by vendor, model, and firmware version, so mixed-firmware fleets stay fully managed and nothing breaks when one device is upgraded independently. The dashboard renders from a form schema the adapter provides, so new devices and new firmware are supported without changing the UI. Three integration levels coexist under one control surface:

- **Fully integrated** — an adapter drives everything through the device's API: bulk operations, validation, change history, cross-vendor parity.
- **Partially integrated** — the adapter unifies part of the device; the rest is exposed through the device's own **web GUI**, tunneled through the platform and gated by the same permissions and audit as any other action. No VPN, no bastion, no port forwarding.
- **Non-integrated (via bridge)** — a device with no usable API or GUI is represented by a bridge appliance and joins the fleet with the same control surface as any other device.

No "unsupported" tier: current, legacy, and closed hardware coexist under one dashboard.

---

## Media preview — one specialized workload (optional)

For devices that serve a media stream (cameras, encoders, media gateways), RG offers in-browser live preview built on the same tunnel — it is a specialization of the general "expose a local service" mechanism, not a separate system.

Preview reuses the **same reverse tunnel the device already holds for control** rather than opening a second connection. The cluster node's local **MediaMTX** ingests the stream straight off the reused tunnel at `{scheme}://{credentials}@127.0.0.1:{tunnel_port}{source_uri}`, republishes it as **WHEP** (WebRTC-HTTP Egress Protocol), and the browser establishes a direct **WebRTC** peer connection for sub-second-latency video. Preview inherits the device's tenant scope from the tunnel allocation record and follows the tunnel automatically on node failover — no new device-side session, no additional credential presentation, no extra port exposed from the device. Devices that don't serve media simply don't use this path; nothing about the platform depends on it.

---

## Security & access control

Security is the architecture, not a feature bolted onto it. Every boundary is an authenticated, encrypted, independently revocable channel.

- **Per-component identity.** Distinct, non-overlapping identities for users, machines (API credentials), devices, and nodes. Device identity is issued by the pairing cluster, bound to a device-generated key, valid only on that cluster. Nodes authenticate each other by asymmetric key, never a shared secret.
- **Two-tier RBAC.** Every authorization decision combines an organization-scope role with an optional per-cluster role override. Built-in roles (owner, administrator, operator, viewer) plus custom roles defined against a resource-typed permission catalog. Business logic declares the permission it needs; the platform resolves the caller's effective role per request, so a grant or revocation takes effect on the very next request — no stale-permission window.
- **True multi-tenancy.** On shared clusters, isolation is enforced at the query, audit, and resource-allocation layers, not just the UI. Cross-tenant lookups return "not found," not "forbidden," to prevent enumeration.
- **Transport.** Modern TLS for user traffic; mutual authentication device-to-cluster; encrypted private mesh between nodes with per-node keys.
- **Credential hygiene.** Short-lived cluster-scoped user tokens, automatic rotation of device tokens and transport certificates through the heartbeat channel, individual revocation by unique identifier, one-time-reveal API secrets stored only as irreversible hashes.
- **Audit at every layer.** Authentications, permission changes, pairing approvals, config changes, adapter invocations, firmware upgrades, native-GUI access, and secret access captured with actor, target, outcome, source, and correlation ID. High-frequency heartbeat/telemetry is deliberately excluded so the audit stays investigation-grade.

---

## Deployment models

Three orthogonal dimensions, combined freely:

- **Hosting** — cloud-hosted or on-premises.
- **Connectivity** — connected (syncs with the cloud layer for global visibility) or air-gapped (fully isolated; local identity, RBAC, and audit, with safe reconciliation if later reconnected).
- **Tenancy** — shared (multi-tenant) or private (single-tenant, adding a physical isolation boundary).

A single customer can run one connected cloud cluster for the general fleet, an on-prem connected cluster for a regulated workload, and an on-prem air-gapped cluster for a classified site — all from the same operator identities and the same permission model. The operating principle is **local-first, cloud-when-available**: devices depend on their cluster, clusters operate autonomously, and the cloud layer is an enhancement, not a dependency.

---

## Who runs on RG

Any operator with a distributed fleet of IP hardware that has a web dashboard or an API and lives on networks they don't fully control:

Network and infrastructure operations (switches, routers, firewalls, access points, PDUs, UPS units across many sites) · industrial and OT (PLCs, RTUs, gateways, environmental and process sensors behind restrictive plant networks) · physical security and surveillance (multi-site, mixed-vendor camera and NVR fleets with strict tenant boundaries and long-retention audit) · enterprise IoT (appliances and control hardware deployed across geographies behind corporate firewalls) · broadcast, production, and media (encoder fleets, media gateways, REMI remote-contribution points, and IRL rigs) · drone and unmanned platforms (telemetry and command-and-control over constrained uplinks, with offline survivability) · managed service providers and integrators (strict per-customer isolation, white-label operation) · regulated and sovereign environments (defense, public safety, healthcare, finance requiring on-prem or fully isolated deployment).

---

## Glossary (quick reference)

**RG Platform** — the whole system. **RG Bridge** — the component (agent or appliance) that carries a device's traffic to a cluster. **RG Cluster** — a regional, autonomous execution plane made of interchangeable nodes. **Device shadow** — the canonical vendor-neutral representation of a device, built from its API. **Adapter** — per-vendor/model/firmware translation behind the shadow. **Native GUI access** — the device's own web dashboard, tunneled through the platform and access-controlled. **Form factor** — `agent` (embedded on device) or `appliance` (standalone bridge). **Attachment state** — `never_attached` / `attached` / `detached`. **CGNAT** — carrier-grade NAT, the network condition RG's outbound-only model is built to traverse.

---

Docs: [docs.remotegeni.us](https://docs.remotegeni.us) · Site: [remotegeni.us](https://remotegeni.us)
