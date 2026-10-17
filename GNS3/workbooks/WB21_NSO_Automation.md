# WB21 — NSO & Automation (Streaming Telemetry)

**Platform:** IOS-XR (EVE-NG) + external **Cisco NSO** server (local VM/container with XR NEDs)
**Scope:** NSO device onboarding + L3VPN service orchestration across the lab PEs; model-driven telemetry on XR nodes
**Level:** CCIE SP
**Prerequisite:** WB16-WB20 XR nodes reachable + manageable (SSH); WB05 L3VPN design (VRF/RD/RT policy)

> **Platform note:** NSO is an **external orchestrator** (CFS/RFS, YANG) — you need an NSO install plus
> IOS-XR NEDs and SSH-manageable XR nodes. Model-driven telemetry (gRPC) is **IOS-XR / IOS-XE 16+** only.
> Neither runs on the GNS3 7200/IOS 15.2 lab — use the EVE-NG XR nodes.
>
> **Format:** Tasks only. No configurations provided — you derive the config/CLI.
> Work **progressively**: verify each section before advancing.
> **Snapshot** NSO (`ncs-backup`) and the devices before/after each section.

> **Reference devices (manage from NSO):** PEs R1 (150.1.31.31), R2 (150.1.2.2), R14 (150.2.14.14), R22 (150.3.22.22), R23 (150.3.23.23); RRs R7/R8/R19/R30.
> NSO runs on a management host reachable to all device mgmt addresses.
> Telemetry collector (TIG/Pipeline) reachable from the XR nodes.

---

## Pre-flight

- [ ] NSO installed and running (`ncs --status` / NSO CLI reachable)
- [ ] IOS-XR NED installed and loaded in NSO (`show packages package … oper-status`)
- [ ] XR devices reachable over SSH from the NSO host; mgmt addresses documented
- [ ] **Snapshot:** NSO `ncs-backup` tagged `WB21-00-baseline`

---

## Section 1 — NSO Device Onboarding [5 points]

Onboard at least two PEs (e.g. R1 and R14) into NSO.

- [ ] **Task 1.1 [1 point]** — In NSO config mode, create each device under `devices device <name>`: set **address** (mgmt IP) and **port**.
- [ ] **Task 1.2 [1 point]** — Create/assign an **authgroup** (`devices authgroups group <name>` with umap + remote-name/password) and reference it on the device (`authgroup <name>`).
- [ ] **Task 1.3 [1 point]** — Set the **ned-id** (device-type cli/netconf ned-id for the IOS-XR NED).
- [ ] **Task 1.4 [1 point]** — **Unlock** the device, run **ssh fetch-host-keys**, then **sync-from** to pull the running config into NSO's CDB.
- [ ] **Task 1.5 [1 point]** — Confirm the device state is reachable and in-sync.
- [ ] **Task 1.6 [0 points — verification]** — Confirm CDB holds the device config.

**Verify / Snapshot**
- [ ] `devices device <name> connect` — result true
- [ ] `devices device <name> check-sync` — in-sync
- [ ] `show devices device <name> config … | nomore` — populated after sync-from
- [ ] **Snapshot:** `WB21-01-onboard`

---

## Section 2 — Apply Device Template (XR-base) [3 points]

- [ ] **Task 2.1 [2 points]** — Create a **device template** named **XR-base** (`devices template XR-base`) carrying common baseline config (e.g. logging, SNMP/NTP, domain, banner — pick a consistent baseline). Model it against the XR NED paths.
- [ ] **Task 2.2 [1 point]** — Apply the template to the onboarded devices (`devices device <name> apply-template template-name XR-base`) and **commit**. Confirm the baseline lands on the devices.
- [ ] **Task 2.3 [0 points — verification]** — `check-sync` after apply (should remain in-sync; CDB == device).

**Verify / Snapshot**
- [ ] `devices template XR-base` present with expected nodes
- [ ] `devices device <name> apply-template` dry-run then commit — no errors
- [ ] On device: baseline config present
- [ ] **Snapshot:** `WB21-02-template`

---

## Section 3 — Deploy L3VPN Service (CCIEl3vpn) [6 points]

Model-driven service that provisions a customer VRF across two PEs.

- [ ] **Task 3.1 [1 point]** — Confirm/load a service package exposing an **`CCIEl3vpn`** service (CFS). (If authoring: a YANG service model with leaves for service-name, VRF/RD/RT, PE+interface, PE-CE details; a template mapping to XR NED paths. If pre-built, locate it under `services`.)
- [ ] **Task 3.2 [2 points]** — Create a **service instance**: `services CCIEl3vpn <instance>` with the VRF name, RD/RT (per WB05 policy), and the two endpoints (e.g. R1 + R14) with PE-CE interface/addressing.
- [ ] **Task 3.3 [1 point]** — Run **`commit dry-run`** and review the per-device native config NSO will push (RFS output). Confirm it matches the intended VRF/BGP/interface config.
- [ ] **Task 3.4 [1 point]** — **Commit** the service and confirm both PEs receive the VRF config and the service is deployed.
- [ ] **Task 3.5 [1 point]** — Confirm **FASTMAP** behavior: modify one service leaf (e.g. add an RT) and `commit dry-run` — only the delta should be generated.
- [ ] **Task 3.6 [0 points — verification]** — End-to-end: the customer VRF is reachable across the two PEs.

**Verify / Snapshot**
- [ ] `commit dry-run outformat native` — correct per-device delta
- [ ] `show services CCIEl3vpn <instance> modified devices` — both PEs
- [ ] On PEs: `show running-config vrf <name>` + VPNv4 route exchange
- [ ] **Snapshot:** `WB21-03-l3vpn-service`

---

## Section 4 — Service Troubleshooting (sync / reconcile) [4 points]

- [ ] **Task 4.1 [1 point]** — **check-sync** a managed device after an **out-of-band** change (manually edit VRF config directly on a PE). Confirm NSO reports **out-of-sync**.
- [ ] **Task 4.2 [1 point]** — Run **`compare-config`** to see the CDB-vs-device diff.
- [ ] **Task 4.3 [2 points]** — Reconcile: choose and justify one of —
  - `devices device <name> sync-from` (accept the device change into CDB), OR
  - `devices device <name> sync-to` (overwrite the device from CDB), OR
  - service **re-deploy reconcile** (`services … re-deploy reconcile`) to restore service intent while absorbing non-service OOB config.
  Confirm the device returns in-sync and the service is intact.
- [ ] **Task 4.4 [0 points — verification]** — `check-sync` → in-sync post-reconcile.

**Verify / Snapshot**
- [ ] `devices device <name> check-sync` — out-of-sync then in-sync
- [ ] `devices device <name> compare-config` — shows the OOB delta
- [ ] `services CCIEl3vpn <instance> re-deploy reconcile dry-run` — intended result
- [ ] **Snapshot:** `WB21-04-reconcile`

---

## Section 5 — Streaming Telemetry (gRPC dial-out) [5 points]

Configured **on the XR devices** (not NSO) — model-driven telemetry to a collector.

- [ ] **Task 5.1 [2 points]** — On an XR PE (e.g. R14), define a **sensor-group** (`telemetry model-driven sensor-group <name>`) with sensor-paths (e.g. interface counters, BGP neighbor state, IS-IS). Document the YANG paths used.
- [ ] **Task 5.2 [1 point]** — Define a **destination-group** (collector IP + port + encoding self-describing-gpb/kvgpb + protocol gRPC) and a **subscription** binding sensor-group → destination-group with a **sample-interval**.
- [ ] **Task 5.3 [1 point]** — Configure **gRPC dial-out** (`telemetry model-driven` subscription referencing the destination). Confirm the session establishes to the collector.
- [ ] **Task 5.4 [1 point]** — Configure the **TPA (third-party / telemetry path) address** / `tpa` VRF-based management source where telemetry egresses a mgmt VRF (`tpa vrf <mgmt> address-family ipv4 … source`). Document why TPA is used (telemetry out of a mgmt VRF).
- [ ] **Task 5.5 [0 points — verification]** — Confirm telemetry rows arrive at the collector.

**Verify / Snapshot**
- [ ] `show telemetry model-driven sensor-group <name>` — paths resolved
- [ ] `show telemetry model-driven subscription <name>` — state ACTIVE, destination connected
- [ ] Collector side: data points received
- [ ] **Snapshot:** `WB21-05-telemetry`

---

## Section 6 — Troubleshooting Drills [4 points]

- [ ] **Drill 6.1 — Device unreachable in NSO [1 point]:** wrong authgroup/credentials. `connect` fails. Diagnose and fix.
- [ ] **Drill 6.2 — Service commit error [2 points]:** supply an invalid leaf (RD format / missing endpoint) in the CCIEl3vpn instance. Commit fails validation. Diagnose from the NSO error and correct.
- [ ] **Drill 6.3 — Telemetry subscription down [1 point]:** collector unreachable / wrong encoding. Subscription not ACTIVE. Diagnose with `show telemetry model-driven subscription` and fix.
- [ ] **Snapshot:** `WB21-06-tshoot`

**Required commands (fluency):**
- [ ] `devices device <name> check-sync | compare-config | sync-from | sync-to`
- [ ] `commit dry-run [outformat native]`
- [ ] `services CCIEl3vpn <instance> re-deploy reconcile`
- [ ] `show telemetry model-driven subscription`

---

## Section 7 — Ansible Playbooks for IOS-XR [4 points]

- [ ] **Task 7.1 [2 points]** — Write an **Ansible playbook** that collects show commands from **all XR devices** (`show isis neighbors`, `show bgp summary`) and saves each device's output to a per-host file. Use the **`cisco.iosxr`** collection (`iosxr_command` module); drive the device list from an inventory group and register the output to write with `copy`/`template`.
- [ ] **Task 7.2 [2 points]** — Write an Ansible playbook that **deploys a standard config template** (NTP, logging, banner) to all XR devices. Use the **`iosxr_config`** module with a `src:` Jinja2 template so the baseline is rendered per device and pushed idempotently.

**Verify / Snapshot**
- [ ] `ansible-playbook … --check` (dry run) succeeds against the XR inventory
- [ ] Per-host output files created (collect playbook)
- [ ] On devices: NTP/logging/banner present after the config playbook
- [ ] **Snapshot:** `WB21-07-ansible`

---

## Section 8 — Python Scripts with IOS-XR [3 points]

- [ ] **Task 8.1 [2 points]** — Write a **Python script** using **netmiko** or **ncclient (NETCONF)** that connects to an XR device, retrieves the running config for `router isis`, and **parses** the output (e.g. extract net/area, interfaces, metrics).
- [ ] **Task 8.2 [1 point]** — Write a Python script that **pushes a VRF configuration** to a PE using **NETCONF** with a **YANG model payload** (`Cisco-IOS-XR-infra-rsi-cfg` / `l3vpn` models) via `ncclient edit-config`.

**Verify / Snapshot**
- [ ] Script output shows parsed IS-IS fields
- [ ] On the PE: VRF created after the NETCONF edit-config
- [ ] `show configuration commit list` reflects the programmatic commit
- [ ] **Snapshot:** `WB21-08-python`

---

## Section 9 — gNMI (gRPC Network Management Interface) [3 points]

- [ ] **Task 9.1 [2 points]** — Configure the **gNMI server** on IOS-XR (`grpc` → `port`, `tls` / certificate, `no-tls` only for lab). Use a **gNMI client** (`gnmic` or similar) to **Subscribe** to interface counters (STREAM / SAMPLE mode) and confirm telemetry rows stream in.
- [ ] **Task 9.2 [1 point]** — In the attempt notes, compare **gNMI vs gRPC telemetry**: dial-in (collector initiates, gNMI Subscribe/Get/Set) vs dial-out (device initiates to a configured destination); the gNMI RPC set (Capabilities / Get / Set / Subscribe) and when each is used.

**Verify / Snapshot**
- [ ] `show grpc` — server UP, port/tls state
- [ ] `gnmic … subscribe` — interface counter updates received
- [ ] **Snapshot:** `WB21-09-gnmi`

---

## Section 10 — Secure ZTP (Zero Touch Provisioning) [3 points]

- [ ] **Task 10.1 [2 points]** — Describe/configure **ZTP** on IOS-XR — the **iPXE boot** process, **DHCP options** (option 67 for the script/boot-file URL, option 150/66 for TFTP), and a **ZTP script** (Python or shell) that applies base config on first boot. Document the end-to-end first-boot flow.
- [ ] **Task 10.2 [1 point]** — **Secure ZTP** — signed scripts, certificate validation, **owner certificate** / ownership voucher (RFC 8572 Secure ZTP). In the notes, explain the **security risks of unsecured ZTP** (rogue DHCP/boot server, malicious script injection, device onboarding to an attacker's config).

**Verify / Snapshot**
- [ ] ZTP flow documented (iPXE → DHCP → script → commit)
- [ ] `show ztp log` / `show ztp status` — provisioning result
- [ ] **Snapshot:** `WB21-10-ztp`

---

## Section 11 — IOS-XR Application Hosting [2 points]

- [ ] **Task 11.1 [1 point]** — Document the **concepts of XR app hosting** — native apps vs **docker containers** on XR, third-party **LXC** containers, and the **East-West** API between a hosted app and XR (shared management/data interfaces).
- [ ] **Task 11.2 [1 point]** — Configure a simple **hosted application** (e.g. a monitoring script in a container): `app-hosting appid <name>`, `app-hosting activate`, `app-hosting start`. Verify: `show app-hosting`.

**Verify / Snapshot**
- [ ] `show app-hosting list` / `show app-hosting detail` — app RUNNING
- [ ] **Snapshot:** `WB21-11-apphosting`

---

## Section 12 — RESTCONF [2 points]

- [ ] **Task 12.1 [1 point]** — Enable **RESTCONF** on IOS-XR (`restconf` + the HTTP/HTTPS server). Use **curl** or Postman to **GET** the running config via RESTCONF (`https://<device>/restconf/data/Cisco-IOS-XR-...`), confirming a 200 with the YANG-modelled JSON/XML.
- [ ] **Task 12.2 [1 point]** — Use RESTCONF to **PUT/PATCH** a configuration change (e.g. add a loopback interface) and confirm it commits on the device.

**Verify / Snapshot**
- [ ] `curl -k … GET` returns running config (RESTCONF)
- [ ] `curl -k … PUT/PATCH` creates the loopback; `show run interface Loopback<n>` confirms
- [ ] **Snapshot:** `WB21-12-restconf`

---

## Scoring Summary

| Section | Max Points | Difficulty Breakdown |
|---------|-----------|---------------------|
| Section 1: NSO device onboarding | 5 | Intermediate: address/authgroup/ned/sync-from |
| Section 2: Device template (XR-base) | 3 | Intermediate: template + apply |
| Section 3: L3VPN service (CCIEl3vpn) | 6 | CCIE-level: CFS/RFS, dry-run, FASTMAP |
| Section 4: Troubleshooting sync/reconcile | 4 | CCIE-level: check-sync, compare, reconcile |
| Section 5: Streaming telemetry (gRPC) | 5 | CCIE-level: sensor/destination/subscription, TPA |
| Section 6: Troubleshooting Drills | 4 | CCIE-level: break-and-fix |
| Section 7: Ansible playbooks for XR | 4 | Intermediate: cisco.iosxr collect + config |
| Section 8: Python scripts with XR | 3 | Intermediate: netmiko/ncclient + NETCONF/YANG |
| Section 9: gNMI | 3 | CCIE-level: gNMI server + subscribe |
| Section 10: Secure ZTP | 3 | CCIE-level: iPXE/DHCP + signed scripts |
| Section 11: XR application hosting | 2 | Intermediate: app-hosting container |
| Section 12: RESTCONF | 2 | Intermediate: GET + PUT/PATCH |
| **Total** | **44** | |

**Scoring rules:**
- Configured from memory, no help: **full points**
- Configured after using `?` / NSO completion: **2/3 of task points (round up)**
- Configured after checking solution: **1/3 of task points (round up)**
- Skipped: **0 points**
- Verification tasks: **0 points** (but you must do them to catch errors)

**Rating:**
```
38-44 points:  CCIE ready for this topic
30-37 points:  Strong, minor gaps to review
20-29 points:  Needs more practice
Below 20:      Review the topic theory before retrying
```

---

## Completion Criteria

- [ ] Devices onboarded (address/authgroup/ned-id/fetch-host-keys/sync-from) and in-sync
- [ ] XR-base device template applied and committed
- [ ] CCIEl3vpn service deployed across two PEs with dry-run + FASTMAP delta verified
- [ ] OOB change detected (check-sync/compare-config) and reconciled (re-deploy reconcile)
- [ ] gRPC dial-out telemetry (sensor-group/destination/subscription) + TPA address active
- [ ] All troubleshooting drills diagnosed and recovered
- [ ] Ansible playbooks: collect (cisco.iosxr) + config template deploy
- [ ] Python: netmiko/ncclient parse IS-IS + NETCONF/YANG VRF push
- [ ] gNMI server + subscribe to interface counters
- [ ] Secure ZTP flow documented (iPXE/DHCP + signed script risks)
- [ ] XR application hosting container RUNNING
- [ ] RESTCONF GET + PUT/PATCH (add loopback) verified
- [ ] **Final snapshot:** NSO `ncs-backup` tagged `WB21-FINAL`

---

## Attempt History

### Attempt 1 — (date)

| Task | Max | Score | Notes |
|------|-----|-------|-------|
| **Section 1: Device onboarding** | **5** | | |
| 1.1 Address + port | 1 | | |
| 1.2 Authgroup | 1 | | |
| 1.3 ned-id | 1 | | |
| 1.4 unlock/fetch-host-keys/sync-from | 1 | | |
| 1.5 Reachable + in-sync | 1 | | |
| **Section 2: Device template** | **3** | | |
| 2.1 XR-base template | 2 | | |
| 2.2 Apply + commit | 1 | | |
| **Section 3: L3VPN service** | **6** | | |
| 3.1 Service package | 1 | | |
| 3.2 Service instance | 2 | | |
| 3.3 commit dry-run native | 1 | | |
| 3.4 Commit + deploy | 1 | | |
| 3.5 FASTMAP delta | 1 | | |
| **Section 4: Reconcile** | **4** | | |
| 4.1 check-sync (OOB) | 1 | | |
| 4.2 compare-config | 1 | | |
| 4.3 re-deploy reconcile | 2 | | |
| **Section 5: Telemetry** | **5** | | |
| 5.1 sensor-group | 2 | | |
| 5.2 destination + subscription | 1 | | |
| 5.3 gRPC dial-out | 1 | | |
| 5.4 TPA address | 1 | | |
| **Section 6: Troubleshooting** | **4** | | |
| 6.1 Device unreachable | 1 | | |
| 6.2 Service commit error | 2 | | |
| 6.3 Telemetry sub down | 1 | | |
| **Section 7: Ansible** | **4** | | |
| 7.1 Collect playbook | 2 | | |
| 7.2 Config template playbook | 2 | | |
| **Section 8: Python** | **3** | | |
| 8.1 Parse IS-IS (netmiko/ncclient) | 2 | | |
| 8.2 NETCONF VRF push | 1 | | |
| **Section 9: gNMI** | **3** | | |
| 9.1 gNMI server + subscribe | 2 | | |
| 9.2 gNMI vs gRPC compare | 1 | | |
| **Section 10: Secure ZTP** | **3** | | |
| 10.1 ZTP flow | 2 | | |
| 10.2 Secure ZTP + risks | 1 | | |
| **Section 11: App hosting** | **2** | | |
| 11.1 Concepts | 1 | | |
| 11.2 Hosted app running | 1 | | |
| **Section 12: RESTCONF** | **2** | | |
| 12.1 GET running config | 1 | | |
| 12.2 PUT/PATCH loopback | 1 | | |
| **TOTAL** | **44** | | |
