# CCIE SP Workbook 16 — Automation & Assurance

**Domain:** 6 — Automation & Assurance (20% of the CCIE SP lab)
**Platform:** IOS-XRv 9000 + NSO VM
🔴 **CCIE Prep Platform:** EVE-NG — see `../00_EVENG_Topology.md`
**Topology:** All 3 ISPs (Emerald + Gold + Garnet) = 20 XRv routers, plus a Linux NSO VM. All devices reachable via the OOB management network. NSO/collector/workstation share the mgmt subnet (assume `10.0.0.0/24`, NSO/collector = `10.0.0.100`).

**Format:** Each task is **Question → Solution → Verification**. All device syntax is IOS-XR unless noted. Python snippets target Python 3.9+.

---

## Section 1: NETCONF / YANG

### Task 1.1 — Enable NETCONF on all 20 routers

**Question:** Enable NETCONF-over-SSH (port 830) on every XRv router so NSO, ncclient, and gNMI collectors can reach them. What is the minimum config, and how do you confirm the agent is listening?

**Solution:**
```
! IOS-XR — per router
configure
 ssh server v2
 ssh server vrf default
 ssh server netconf vrf default        ! opens TCP/830 in the mgmt VRF
 ssh server netconf port 830           ! default; explicit for clarity
 netconf-yang agent ssh                ! enables the NETCONF/YANG agent
 crypto key generate rsa               ! (exec mode, once) if no host key yet
commit
```
Roll this out to all 20 routers with a loop (see Section 4) or an NSO template. Key points:
- `netconf-yang agent ssh` binds the agent to the SSH server — without it the TCP session opens but the `<hello>` never arrives.
- The SSH server must exist in the VRF you connect through (`vrf default` for mgmt in this topology).

**Verification:**
```
RP/0/RP0/CPU0:E-R1# show netconf-yang statistics
RP/0/RP0/CPU0:E-R1# show ssh session-details
RP/0/RP0/CPU0:E-R1# show netconf-yang clients
```
From the NSO/workstation, prove the hello handshake:
```bash
ssh -p 830 -s cisco@10.1.1.1 netconf
# You should receive an XML <hello> listing capabilities and <session-id>.
```

---

### Task 1.2 — YANG model navigation (native + OpenConfig)

**Question:** You must know which YANG model to query. How do you list the models a router advertises, and what is the difference between the Cisco-IOS-XR native models and OpenConfig?

**Solution:**
- **Native (`Cisco-IOS-XR-*`)** models map 1:1 to XR CLI. Config models end in `-cfg`, operational models end in `-oper`. Examples:
  - `Cisco-IOS-XR-ipv4-bgp-cfg` (BGP config)
  - `Cisco-IOS-XR-clns-isis-cfg` (IS-IS config)
  - `Cisco-IOS-XR-infra-statsd-oper` (interface counters, oper)
- **OpenConfig (`openconfig-*`)** models are vendor-neutral, used for multi-vendor tooling and gNMI. Examples:
  - `openconfig-interfaces`, `openconfig-network-instance` (BGP/IS-IS live under network-instance protocols), `openconfig-bgp`.
- On XR, OpenConfig is a **mapped overlay** onto native; a config pushed via OpenConfig is translated to native under the hood.

List advertised capabilities from the `<hello>` (all supported models appear as capability URIs), or with pyang locally:
```bash
pyang -f tree Cisco-IOS-XR-clns-isis-cfg.yang | head -40
pyang -f tree openconfig-network-instance.yang
```

**Verification:**
```
RP/0/RP0/CPU0:E-R1# show netconf-yang trace           ! shows model access during a session
```
```python
# ncclient — dump the capability list
from ncclient import manager
with manager.connect(host="10.1.1.1", port=830, username="cisco",
                     password="cisco123", hostkey_verify=False,
                     device_params={"name": "iosxr"}) as m:
    for cap in m.server_capabilities:
        if "isis" in cap or "openconfig" in cap:
            print(cap)
```

---

### Task 1.3 — Retrieve running config with ncclient

**Question:** Use the Python `ncclient` library to pull E-R1's IS-IS configuration via NETCONF and print the XML.

**Solution:**
```python
#!/usr/bin/env python3
from ncclient import manager
import xml.dom.minidom as minidom

ISIS_FILTER = """
<filter>
  <isis xmlns="http://cisco.com/ns/yang/Cisco-IOS-XR-clns-isis-cfg"/>
</filter>
"""

def get_isis(host):
    with manager.connect(host=host, port=830, username="cisco",
                         password="cisco123", hostkey_verify=False,
                         device_params={"name": "iosxr"},
                         allow_agent=False, look_for_keys=False) as m:
        reply = m.get_config(source="running", filter=ISIS_FILTER)
        print(minidom.parseString(reply.xml).toprettyxml(indent="  "))

if __name__ == "__main__":
    get_isis("10.1.1.1")
```

**Verification:**
- The XML returned should contain `<instance><instance-name>1</instance-name>` and the interface/metric leaves.
- Cross-check on the box:
```
RP/0/RP0/CPU0:E-R1# show running-config router isis
```

---

### Task 1.4 — Push an IS-IS metric change with edit-config (candidate → commit)

**Question:** Change the IS-IS metric on E-R1's `GigabitEthernet0/0/0/0` from 10 to 100 using NETCONF `edit-config` against the **candidate** datastore, then commit. Demonstrate you understand the XR candidate→commit model.

**Solution:**
XR NETCONF is transactional: `edit-config target=candidate` stages the change; `commit` applies it atomically (identical to CLI `commit`). If you never commit, the candidate is discarded on disconnect.
```python
#!/usr/bin/env python3
from ncclient import manager

EDIT = """
<config>
  <isis xmlns="http://cisco.com/ns/yang/Cisco-IOS-XR-clns-isis-cfg">
    <instances>
      <instance>
        <instance-name>1</instance-name>
        <interfaces>
          <interface>
            <interface-name>GigabitEthernet0/0/0/0</interface-name>
            <interface-afs>
              <interface-af>
                <af-name>ipv4</af-name>
                <saf-name>unicast</saf-name>
                <interface-af-data>
                  <metrics>
                    <metric>
                      <level>not-set</level>
                      <metric>100</metric>
                    </metric>
                  </metrics>
                </interface-af-data>
              </interface-af>
            </interface-afs>
          </interface>
        </interfaces>
      </instance>
    </instances>
  </isis>
</config>
"""

with manager.connect(host="10.1.1.1", port=830, username="cisco",
                     password="cisco123", hostkey_verify=False,
                     device_params={"name": "iosxr"},
                     allow_agent=False, look_for_keys=False) as m:
    m.edit_config(target="candidate", config=EDIT)
    # optional: m.validate(source="candidate")
    m.commit()
    print("committed")
```
Equivalent target CLI being generated:
```
router isis 1
 interface GigabitEthernet0/0/0/0
  address-family ipv4 unicast
   metric 100
```

**Verification:**
```
RP/0/RP0/CPU0:E-R1# show running-config router isis 1 interface GigabitEthernet0/0/0/0
RP/0/RP0/CPU0:E-R1# show isis interface GigabitEthernet0/0/0/0 | include Metric
RP/0/RP0/CPU0:E-R1# show configuration commit list          ! candidate→commit history
```
Confirm neighbors recomputed the path (metric now 100):
```
RP/0/RP0/CPU0:E-R1# show isis route
```

---

## Section 2: gRPC / gNMI Model-Driven Telemetry

### Task 2.1 — Enable gRPC on XRv (port 57400)

**Question:** Enable the gRPC server on all XRv routers on port 57400 so a collector can dial-in (gNMI) and the router can dial-out (MDT).

**Solution:**
```
configure
 grpc
  port 57400
  no-tls              ! lab only — production should use tls with certs
  address-family ipv4
commit
```

**Verification:**
```
RP/0/RP0/CPU0:E-R1# show grpc status
RP/0/RP0/CPU0:E-R1# show grpc statistics
```
From the collector: `netstat -an | grep 57400`, or a gNMI capabilities probe:
```bash
gnmic -a 10.1.1.1:57400 -u cisco -p cisco123 --insecure capabilities
```

---

### Task 2.2 — Dial-out vs dial-in (concept + when to use)

**Question:** Explain MDT **dial-out** vs **dial-in** and identify which one the router initiates.

**Solution:**
- **Dial-out (router pushes):** The *router* opens the session to the collector and streams data. Configured entirely on the router (sensor-group + subscription + destination-group). Best for steady production telemetry pipelines; survives collector restarts poorly (router must reconnect).
- **Dial-in (collector pulls / gNMI SUBSCRIBE):** The *collector* opens a gNMI session to the router and subscribes. No telemetry config on the router beyond enabling gRPC. Best for ad-hoc/on-demand queries and multi-vendor tooling.

| | Dial-out | Dial-in (gNMI) |
|---|---|---|
| Session initiator | Router | Collector |
| Router config | sensor/subscription/destination | just `grpc` |
| Transport | gRPC/TCP or UDP | gRPC/TCP (57400) |
| Use case | Always-on pipeline | On-demand / multi-vendor |

**Verification:** Conceptual — validated by Tasks 2.3–2.4.

---

### Task 2.3 — Configure dial-out telemetry (sensor-path, subscription, destination)

**Question:** On E-R1 configure a dial-out subscription that streams **interface stats**, **BGP neighbor state**, and **IS-IS adjacency** to the collector `10.0.0.100:57500` every 10 seconds.

**Solution:**
```
configure
 telemetry model-driven
  destination-group DG-COLLECTOR
   address-family ipv4 10.0.0.100 port 57500
    encoding self-describing-gpb
    protocol grpc no-tls
   !
  !
  sensor-group SG-CORE
   sensor-path Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters
   sensor-path Cisco-IOS-XR-ipv4-bgp-oper:bgp/instances/instance/instance-active/default-vrf/neighbors/neighbor
   sensor-path Cisco-IOS-XR-clns-isis-oper:isis/instances/instance/neighbors/neighbor
  !
  subscription SUB-CORE
   sensor-group-id SG-CORE sample-interval 10000    ! milliseconds = 10s
   destination-id DG-COLLECTOR
  !
 !
commit
```
Notes:
- `sample-interval` is in **milliseconds**. `0` = event-driven (on-change) where the model supports it.
- `self-describing-gpb` embeds keys/paths so the collector needs no proto compilation.

**Verification:**
```
RP/0/RP0/CPU0:E-R1# show telemetry model-driven subscription SUB-CORE
RP/0/RP0/CPU0:E-R1# show telemetry model-driven subscription SUB-CORE internal
RP/0/RP0/CPU0:E-R1# show telemetry model-driven sensor-group SG-CORE
RP/0/RP0/CPU0:E-R1# show telemetry model-driven destination DG-COLLECTOR
```
Look for subscription state `Active` and destination state `Active`. On the collector, confirm data every ~10s:
```bash
# pipeline / telegraf / gnmic collector shows counters arriving
gnmic --config gnmic.yaml subscribe   # or watch pipeline logs
```

---

### Task 2.4 — Dial-in gNMI subscribe

**Question:** From the workstation, use gNMI SUBSCRIBE (dial-in) to watch BGP neighbor state on E-R1 at a 5s sample interval, then trigger a change.

**Solution:**
```bash
gnmic -a 10.1.1.1:57400 -u cisco -p cisco123 --insecure subscribe \
  --path "openconfig-network-instance:network-instances/network-instance[name=default]/protocols/protocol/bgp/neighbors/neighbor/state" \
  --sample-interval 5s --mode stream --stream-mode sample
```
Then bounce a session to observe the state transition in the stream:
```
RP/0/RP0/CPU0:E-R1(config)# router bgp 65001
RP/0/RP0/CPU0:E-R1(config-bgp)# neighbor 10.1.1.2 shutdown
! ... observe Idle in the gNMI stream, then:
RP/0/RP0/CPU0:E-R1(config-bgp)# no neighbor 10.1.1.2 shutdown
```

**Verification:** The gNMI stream shows `session-state` transition `ESTABLISHED → IDLE → ESTABLISHED`. Cross-check:
```
RP/0/RP0/CPU0:E-R1# show bgp neighbor 10.1.1.2 | include BGP state
```

---

## Section 3: NSO (Network Services Orchestrator)

### Task 3.1 — NSO architecture

**Question:** Describe the core NSO components and how a service commit flows through them.

**Solution:**
- **CDB (Configuration Database):** In-memory + on-disk datastore holding NSO's view of device config (device model) and service data (service model). Transactional.
- **NED (Network Element Driver):** Per-platform adapter translating CDB config into device-native config (CLI/NETCONF). e.g. `cisco-iosxr-cli-7.x` or `cisco-iosxr-nc-1.x`.
- **Service package (FASTMAP):** Python/Java + YANG that maps a high-level service intent to device config. NSO computes the minimal diff and can auto-derive un-provisioning (delete = reverse-mapping).
- **Flow:** service commit → service mapping produces device config in CDB (candidate) → NEDs render native config → NSO pushes to all devices in **one distributed transaction** (all-or-nothing).

**Verification:**
```
admin@ncs# show packages package oper-status
admin@ncs# show ncs-state
```

---

### Task 3.2 — Add XRv devices and authgroups

**Question:** Add E-R1 and G-R1 to NSO inventory with the correct NED-id and an authgroup.

**Solution:**
```
$ ncs_cli -C -u admin
admin@ncs# config

! authgroup once
admin@ncs(config)# devices authgroups group XRAUTH
admin@ncs(config-group-XRAUTH)# default-map remote-name cisco remote-password cisco123
admin@ncs(config-group-XRAUTH)# exit

admin@ncs(config)# devices device E-R1
admin@ncs(config-device-E-R1)# address 10.1.1.1
admin@ncs(config-device-E-R1)# port 830
admin@ncs(config-device-E-R1)# authgroup XRAUTH
admin@ncs(config-device-E-R1)# device-type netconf ned-id cisco-iosxr-nc-1.0
admin@ncs(config-device-E-R1)# state admin-state unlocked
admin@ncs(config-device-E-R1)# exit

admin@ncs(config)# devices device G-R1
admin@ncs(config-device-G-R1)# address 10.1.1.5
admin@ncs(config-device-G-R1)# port 830
admin@ncs(config-device-G-R1)# authgroup XRAUTH
admin@ncs(config-device-G-R1)# device-type netconf ned-id cisco-iosxr-nc-1.0
admin@ncs(config-device-G-R1)# state admin-state unlocked
admin@ncs(config-device-G-R1)# commit

! fetch host keys
admin@ncs# devices fetch-ssh-host-keys
```

**Verification:**
```
admin@ncs# show devices list
admin@ncs# devices device E-R1 connect
admin@ncs# devices device E-R1 check-sync         ! expect in-sync (after sync-from)
```

---

### Task 3.3 — sync-from

**Question:** Pull the running config from all devices into CDB.

**Solution:**
```
admin@ncs# devices sync-from
! or per device:
admin@ncs# devices device E-R1 sync-from
```

**Verification:**
```
admin@ncs# show devices device E-R1 config router isis
admin@ncs# devices check-sync           ! all should report in-sync
```

---

### Task 3.4 — L3VPN service package: VRF across E-R1 + G-R1 in one transaction

**Question:** Using an L3VPN service package, provision VRF **CUST-A** on both E-R1 and G-R1 in a single NSO transaction. Preview before committing.

**Solution:**
```
admin@ncs# config
admin@ncs(config)# l3vpn CUST-A
admin@ncs(config-l3vpn-CUST-A)# customer "Customer A"
admin@ncs(config-l3vpn-CUST-A)# route-distinguisher 65000
admin@ncs(config-l3vpn-CUST-A)# endpoint E-R1
admin@ncs(config-endpoint-E-R1)#  device E-R1
admin@ncs(config-endpoint-E-R1)#  interface GigabitEthernet0/0/0/2
admin@ncs(config-endpoint-E-R1)#  ip-address 172.16.1.1 prefix-length 30
admin@ncs(config-endpoint-E-R1)#  exit
admin@ncs(config-l3vpn-CUST-A)# endpoint G-R1
admin@ncs(config-endpoint-G-R1)#  device G-R1
admin@ncs(config-endpoint-G-R1)#  interface GigabitEthernet0/0/0/2
admin@ncs(config-endpoint-G-R1)#  ip-address 172.16.2.1 prefix-length 30
admin@ncs(config-endpoint-G-R1)#  exit

! preview the native config NSO WOULD push (no change yet):
admin@ncs(config-l3vpn-CUST-A)# commit dry-run outformat native

! apply atomically to both PEs:
admin@ncs(config-l3vpn-CUST-A)# commit
```
`commit dry-run outformat native` shows the exact `vrf CUST-A / rd / route-target / interface / router bgp vrf` lines destined for each PE. If either PE fails, the whole transaction rolls back.

**Verification:**
```
admin@ncs# show l3vpn CUST-A
admin@ncs# devices device E-R1 live-status exec show vrf CUST-A
admin@ncs# devices device G-R1 live-status exec show vrf CUST-A
```
On the routers directly:
```
RP/0/RP0/CPU0:E-R1# show vrf CUST-A
RP/0/RP0/CPU0:G-R1# show vrf CUST-A
```

---

### Task 3.5 — NSO rollback (undo last commit)

**Question:** The CUST-A rollout was wrong. Undo the last commit atomically.

**Solution:**
```
admin@ncs# show configuration rollback         ! list rollback files (id + label)
admin@ncs# rollback configuration              ! load most recent rollback into candidate
admin@ncs(config)# commit                       ! apply — removes VRF from both PEs
! or target a specific id:
admin@ncs# rollback configuration 10001
```

**Verification:**
```
admin@ncs# show l3vpn CUST-A                     ! should be gone
admin@ncs# devices device E-R1 live-status exec show vrf CUST-A   ! % No VRF
admin@ncs# devices check-sync
```

---

### Task 3.6 — Compliance check (consistent IS-IS across all PEs)

**Question:** Verify that all PE routers have consistent IS-IS configuration using an NSO compliance report.

**Solution:**
```
! define a compliance template capturing the golden IS-IS baseline
admin@ncs# config
admin@ncs(config)# compliance template ISIS-GOLDEN
admin@ncs(config-template-ISIS-GOLDEN)# ned-id cisco-iosxr-nc-1.0
! (edit ncs:config to pin metric-style wide, net area, address-family, etc.)
admin@ncs(config-template-ISIS-GOLDEN)# commit

admin@ncs(config)# compliance reports report ISIS-CHECK
admin@ncs(config-report-ISIS-CHECK)# device-group ALL-PE
admin@ncs(config-report-ISIS-CHECK)# compliance-template ISIS-GOLDEN
admin@ncs(config-report-ISIS-CHECK)# commit
admin@ncs(config)# exit

admin@ncs# compliance reports report ISIS-CHECK run
```

**Verification:**
```
admin@ncs# show compliance reports report ISIS-CHECK
! Inspect the generated report (HTML/XML) for "violations" / "diff" blocks.
! Zero diffs = all PEs consistent with the golden IS-IS template.
```

---

## Section 4: Python Scripting

### Task 4.1 — Collect show commands from all routers (netmiko)

**Question:** Write a Python script using netmiko to collect `show isis neighbors` and `show interfaces summary` from all 20 routers and save per-device output.

**Solution:**
```python
#!/usr/bin/env python3
from netmiko import ConnectHandler
from concurrent.futures import ThreadPoolExecutor
import os

# 20 XRv routers across 3 ISPs
DEVICES = [{"host": f"10.1.1.{i}"} for i in range(1, 21)]
COMMANDS = ["show isis neighbors", "show interfaces summary"]
OUTDIR = "collected"

def collect(dev):
    params = {
        "device_type": "cisco_xr",
        "host": dev["host"],
        "username": "cisco",
        "password": "cisco123",
        "fast_cli": False,
    }
    try:
        with ConnectHandler(**params) as conn:
            os.makedirs(OUTDIR, exist_ok=True)
            for cmd in COMMANDS:
                out = conn.send_command(cmd)
                fn = f"{OUTDIR}/{dev['host']}_{cmd.replace(' ', '_')}.txt"
                with open(fn, "w") as f:
                    f.write(out)
        return (dev["host"], "ok")
    except Exception as e:
        return (dev["host"], f"FAIL: {e}")

if __name__ == "__main__":
    with ThreadPoolExecutor(max_workers=10) as ex:
        for host, status in ex.map(collect, DEVICES):
            print(f"{host}: {status}")
```

**Verification:** `ls collected/` shows 40 files; spot-check one against the live device output.

---

### Task 4.2 — Parse output (regex + TextFSM) → topology report

**Question:** Parse the collected `show isis neighbors` output to build a neighbor adjacency table. Show both a regex approach and TextFSM (ntc-templates).

**Solution — regex:**
```python
import re, glob

# Example line:  E-R2  Gi0/0/0/0  *PtoP*  Up  23  L2
LINE = re.compile(
    r"^(?P<sys>\S+)\s+(?P<intf>\S+)\s+\S+\s+(?P<state>Up|Init|Down)\s+\d+\s+(?P<level>L1|L2|L1L2)"
)

adjacencies = []
for fn in glob.glob("collected/*_show_isis_neighbors.txt"):
    local = fn.split("/")[-1].split("_")[0]
    for line in open(fn):
        m = LINE.match(line.strip())
        if m:
            adjacencies.append((local, m["sys"], m["intf"], m["state"], m["level"]))

print(f"{'LOCAL':<12}{'NEIGHBOR':<12}{'INTF':<16}{'STATE':<8}{'LEVEL'}")
for a in adjacencies:
    print(f"{a[0]:<12}{a[1]:<12}{a[2]:<16}{a[3]:<8}{a[4]}")
```

**Solution — TextFSM (preferred, robust):**
```python
from netmiko import ConnectHandler

params = {"device_type": "cisco_xr", "host": "10.1.1.1",
          "username": "cisco", "password": "cisco123"}
with ConnectHandler(**params) as conn:
    # use_textfsm returns a list of dicts keyed by template fields
    data = conn.send_command("show isis neighbors", use_textfsm=True)
    for row in data:
        print(row)   # {'system_id': 'E-R2', 'interface': 'Gi0/0/0/0', 'state': 'Up', ...}
```

**Verification:** The printed adjacency table matches `show isis neighbors` counts on each router; every expected core link appears as `Up`.

---

### Task 4.3 — Verify IS-IS adjacency health across all 3 SPs

**Question:** Write a health-check script that flags any router whose IS-IS neighbor count is below expected or has non-`Up` adjacencies.

**Solution:**
```python
#!/usr/bin/env python3
from netmiko import ConnectHandler

# expected neighbor count per router (from topology design)
EXPECTED = {f"10.1.1.{i}": 2 for i in range(1, 21)}
EXPECTED["10.1.1.1"] = 3   # E-R1 is a hub example

def check(host, expected):
    params = {"device_type": "cisco_xr", "host": host,
              "username": "cisco", "password": "cisco123"}
    with ConnectHandler(**params) as conn:
        rows = conn.send_command("show isis neighbors", use_textfsm=True)
    up = [r for r in rows if r.get("state", "").lower().startswith("up")]
    problems = []
    if len(up) < expected:
        problems.append(f"only {len(up)}/{expected} adjacencies UP")
    bad = [r for r in rows if not r.get("state", "").lower().startswith("up")]
    for r in bad:
        problems.append(f"neighbor {r.get('system_id')} state={r.get('state')}")
    return host, ("HEALTHY" if not problems else "; ".join(problems))

if __name__ == "__main__":
    exit_code = 0
    for host, exp in EXPECTED.items():
        h, status = check(host, exp)
        print(f"{h:<14} {status}")
        if status != "HEALTHY":
            exit_code = 1
    raise SystemExit(exit_code)   # non-zero → usable in CI / cron alerting
```

**Verification:** Script prints `HEALTHY` for all 20 routers and exits 0. Break a link (`shutdown` on a core interface) → the affected router reports the degraded count and the script exits non-zero.

---

## Section 5: Zero-Touch Provisioning (ZTP)

### Task 5.1 — ZTP concept + iPXE boot + first-boot Python script

**Question:** Explain the ZTP boot flow on IOS-XR and write a first-boot Python ZTP script that sets hostname, a loopback, and IS-IS.

**Solution — flow:**
1. New XRv boots with no config → ZTP triggers on a management interface.
2. It sends **DHCP** (option 66/67 or option 43/150) → receives an IP plus a **provisioning URL** (script or config file).
3. Optionally **iPXE** chain-boots the image over HTTP/TFTP for bare-metal image install, then ZTP runs the provisioning script.
4. XR runs the fetched script (`#!/usr/bin/env python3` with the on-box `ztp_helper` library) or applies a fetched config file.

**Solution — ZTP Python script (runs on the router at first boot):**
```python
#!/usr/bin/env python3
# /disk0:/ztp/provision.py  — served via DHCP option 67
from ztp_helper import ZtpHelpers
import re

ztp = ZtpHelpers()
ztp.toggle_debug(1)

# derive hostname/loopback from the mgmt IP (e.g., .11 -> R11)
mgmt = ztp.get_ipv4_interfaces()  # helper returns mgmt interface info
node_id = "11"                    # in practice parse from DHCP/serial
hostname = f"R{node_id}"
loop_ip = f"10.255.255.{node_id}"
net = f"49.0001.0000.0000.00{node_id.zfill(2)}.00"

config = f"""
hostname {hostname}
interface Loopback0
 ipv4 address {loop_ip} 255.255.255.255
!
router isis 1
 is-type level-2-only
 net {net}
 address-family ipv4 unicast
  metric-style wide
 !
 interface Loopback0
  passive
  address-family ipv4 unicast
 !
 interface GigabitEthernet0/0/0/0
  point-to-point
  address-family ipv4 unicast
 !
!
"""

result = ztp.xrapply_string(config)   # applies + commits
ztp.syslogger.info(f"ZTP result: {result}")
```

**Verification:**
```
RP/0/RP0/CPU0:R11# show ztp log
RP/0/RP0/CPU0:R11# show running-config hostname
RP/0/RP0/CPU0:R11# show running-config interface Loopback0
RP/0/RP0/CPU0:R11# show isis neighbors
```

---

### Task 5.2 — Secure ZTP (certificate-based)

**Question:** Standard ZTP trusts whatever DHCP hands out. How does Secure ZTP (RFC 8572) harden this?

**Solution:**
- **Secure ZTP (RFC 8572)** replaces plain HTTP/TFTP with authenticated, integrity-protected onboarding:
  - The device ships with an **IDevID** (manufacturer-installed X.509 cert, e.g., Cisco SUDI) proving its identity.
  - The bootstrap server is validated via **TLS server cert** against a trust anchor pre-loaded on the device.
  - Provisioning artifacts (boot image, config, scripts) are delivered as **signed "onboarding information"** — the device verifies the signature before applying, preventing tampered/rogue configs.
  - Supports **ownership vouchers** (RFC 8366) so the device only accepts config from its rightful owner.
- On XR, enable the trust anchor / TLS profile for the bootstrap fetch and rely on the SUDI cert for mutual auth (`crypto ca trustpoint`, TLS profile bound to the ZTP fetch).

**Verification:**
```
RP/0/RP0/CPU0:R11# show crypto ca certificates            ! IDevID/SUDI present
RP/0/RP0/CPU0:R11# show ztp log | include TLS|signature|verify
```
A ZTP attempt against an untrusted (unsigned) bundle must be **rejected** — confirm the failure appears in `show ztp log` and no config is applied.

---

## Section 6: Assurance Tools

### Task 6.1 — Syslog to a central server

**Question:** Configure all routers to send syslog to `10.0.0.100` at severity informational with source loopback0.

**Solution:**
```
configure
 logging 10.0.0.100 vrf default severity info
 logging source-interface Loopback0
 logging trap informational
 logging buffered 10000000
 logging hostnameprefix R11               ! optional, aids parsing at the collector
 service timestamps log datetime msec
commit
```

**Verification:**
```
RP/0/RP0/CPU0:R11# show logging | include 10.0.0.100
RP/0/RP0/CPU0:R11# show logging               ! recent events present
```
On the collector: `tail -f /var/log/syslog` shows events; generate one with an interface flap.

---

### Task 6.2 — SNMP v3 (user / group / view)

**Question:** Configure SNMPv3 with authPriv: view (restrict to system + interfaces MIB), group, and user, sending traps to the NMS.

**Solution:**
```
configure
 snmp-server view V-RESTRICTED 1.3.6.1.2.1.1 included        ! system
 snmp-server view V-RESTRICTED 1.3.6.1.2.1.2 included        ! interfaces
 snmp-server group G-RO v3 priv read V-RESTRICTED
 snmp-server user U-MONITOR G-RO v3 auth sha AuthPass123 priv aes 128 PrivPass123
 snmp-server host 10.0.0.100 traps version 3 priv U-MONITOR
 snmp-server traps snmp linkup
 snmp-server traps snmp linkdown
commit
```
- `v3 priv` = authentication **and** encryption (authPriv). `auth` = auth only, `noauth` = neither.

**Verification:**
```
RP/0/RP0/CPU0:R11# show snmp view
RP/0/RP0/CPU0:R11# show snmp group
RP/0/RP0/CPU0:R11# show snmp users
```
From the NMS:
```bash
snmpwalk -v3 -l authPriv -u U-MONITOR -a SHA -A AuthPass123 -x AES -X PrivPass123 10.1.1.11 1.3.6.1.2.1.1
```

---

### Task 6.3 — NetFlow/IPFIX on PE interfaces + BFD echo mode

**Question:** Enable IPFIX (NetFlow v10) export on PE customer-facing interfaces to `10.0.0.100`, and enable BFD echo mode on a core link for fast liveliness.

**Solution — IPFIX:**
```
configure
 flow exporter-map EXP-IPFIX
  version ipfix
  transport udp 4739
  destination 10.0.0.100
  source Loopback0
 !
 flow monitor-map FMM-INGRESS
  record ipv4
  exporter EXP-IPFIX
  cache timeout active 60
 !
 sampler-map SM-1IN100
  random 1 out-of 100
 !
 interface GigabitEthernet0/0/0/2       ! customer-facing PE interface
  flow ipv4 monitor FMM-INGRESS sampler SM-1IN100 ingress
 !
commit
```

**Solution — BFD echo mode:**
```
configure
 interface GigabitEthernet0/0/0/0        ! core link
  bfd mode ietf
  bfd address-family ipv4 echo minimum-interval 50
  bfd address-family ipv4 minimum-interval 300
  bfd address-family ipv4 multiplier 3
 !
 router isis 1
  interface GigabitEthernet0/0/0/0
   bfd fast-detect ipv4                    ! bind BFD to IS-IS
 !
commit
```
Echo mode loops packets back through the neighbor's data plane for sub-second failure detection without burdening the control plane.

**Verification:**
```
RP/0/RP0/CPU0:E-R1# show flow monitor FMM-INGRESS cache
RP/0/RP0/CPU0:E-R1# show flow exporter EXP-IPFIX
RP/0/RP0/CPU0:E-R1# show bfd session interface GigabitEthernet0/0/0/0 detail
RP/0/RP0/CPU0:E-R1# show bfd session                 ! echo-enabled, state Up
```
Collector: IPFIX flows arrive on UDP/4739. Fail the core link → BFD detects in <150ms and IS-IS reconverges.

---

## Section 7: Troubleshooting

### Task 7.1 — NETCONF session rejected

**Question:** A NETCONF client to PE7:830 either refuses the connection or connects but never returns a `<hello>`. Diagnose and fix.

**Solution — root causes & fixes:**
1. **SSH server not running / wrong VRF** → TCP connect refused.
   ```
   RP/0/RP0/CPU0:PE7# show ssh                       ! any sessions? server up?
   RP/0/RP0/CPU0:PE7# show running-config ssh
   ! fix:
   configure
    ssh server v2
    ssh server vrf default
    ssh server netconf vrf default
   commit
   ```
2. **NETCONF agent not enabled** → SSH connects but subsystem `netconf` fails / no `<hello>`.
   ```
   RP/0/RP0/CPU0:PE7# show running-config netconf-yang agent
   ! fix:
   configure
    netconf-yang agent ssh
   commit
   ```
3. **No SSH host key** → handshake fails.
   ```
   RP/0/RP0/CPU0:PE7# show crypto key mypubkey rsa
   ! fix (exec):  crypto key generate rsa
   ```
4. **AAA / credentials** → `%NETCONF: access denied`. Verify the task-group grants config/read; test with `ssh -p 830 ... -s netconf`.

**Verification:**
```
RP/0/RP0/CPU0:PE7# show netconf-yang clients        ! client now listed
```
```bash
ssh -p 830 -s cisco@10.1.1.7 netconf   # returns <hello> with capabilities + session-id
```

---

### Task 7.2 — Telemetry data not arriving

**Question:** The dial-out subscription `SUB-CORE` on E-R1 is configured but the collector receives nothing. Diagnose and fix.

**Solution — root causes & fixes:**
1. **Subscription not Active / sensor-group empty.**
   ```
   RP/0/RP0/CPU0:E-R1# show telemetry model-driven subscription SUB-CORE
   ! State "NA"/"Paused" → check sensor-group binding and destination-id.
   ```
2. **Wrong / invalid sensor-path** → subscription shows the path in error/`Resolved NO`.
   ```
   RP/0/RP0/CPU0:E-R1# show telemetry model-driven subscription SUB-CORE internal
   ! Look for "sensor path ... State: Resolved" vs "NotResolved".
   ! Fix: correct the model path (oper vs cfg, exact container names).
   configure
    telemetry model-driven
     sensor-group SG-CORE
      no sensor-path <bad-path>
      sensor-path Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters
   commit
   ```
3. **Destination unreachable / wrong port / TLS mismatch.**
   ```
   RP/0/RP0/CPU0:E-R1# show telemetry model-driven destination DG-COLLECTOR
   ! State "Active" expected; "Connection Retries" climbing = network/port/TLS issue.
   ! Verify: ping 10.0.0.100 ; collector listening on 57500 ; protocol grpc no-tls matches collector.
   ```
4. **gRPC not enabled** (for grpc transport) → destination never connects.
   ```
   RP/0/RP0/CPU0:E-R1# show grpc status
   ```
5. **ACL/firewall on mgmt path** dropping the export.

**Verification:**
```
RP/0/RP0/CPU0:E-R1# show telemetry model-driven subscription SUB-CORE   ! State Active
RP/0/RP0/CPU0:E-R1# show telemetry model-driven destination DG-COLLECTOR ! State Active, rows sent increasing
```
Collector now receives self-describing-gpb every 10s.

---

## Final Validation Checklist
```
[ ] NETCONF enabled on all 20 XRv (port 830, agent ssh); <hello> received
[ ] YANG: native (Cisco-IOS-XR-*-cfg/-oper) vs OpenConfig navigation understood
[ ] ncclient get-config pulls IS-IS; edit-config candidate→commit pushes metric 100
[ ] gRPC enabled (57400); dial-out vs dial-in explained
[ ] MDT dial-out: sensor-group + subscription (10s) + destination Active
[ ] gNMI dial-in SUBSCRIBE shows BGP state transitions
[ ] NSO: architecture (CDB/NED/service/FASTMAP) explained
[ ] NSO: devices added, sync-from, all in-sync
[ ] NSO: L3VPN CUST-A on E-R1+G-R1 in one transaction (dry-run native reviewed)
[ ] NSO: rollback removes VRF atomically
[ ] NSO: compliance report confirms consistent IS-IS across PEs
[ ] Python: netmiko collection (20 routers), TextFSM/regex parse, adjacency health-check exits 0
[ ] ZTP: concept + first-boot Python script (hostname/loopback/IS-IS); Secure ZTP (RFC 8572) certs
[ ] Assurance: syslog server, SNMPv3 authPriv, IPFIX export, BFD echo mode
[ ] Troubleshooting: NETCONF reject (SSH/agent/hostkey) fixed; telemetry silence (path/dest/grpc) fixed
```

---

## Section 8: NSO Service Package Development (CCIE Exam Critical)

### Task 8.1 — NSO dry-run (commit dry-run outformat native)

**Question:** You are about to push a VRF change via NSO to 3 PEs. Before committing, preview exactly what CLI NSO will send to each device. What command do you use, and what's the difference between `outformat native` and `outformat xml`?

**Solution:**

```
admin@ncs# services l3vpn CUST_A pe [ E-R1 G-R1 Gar-R1 ]
admin@ncs# commit dry-run outformat native
```

Output shows the exact IOS-XR CLI that would be pushed to each device — VRF config, BGP neighbor, interface assignment. `outformat native` = device CLI (what you'd type manually). `outformat xml` = YANG XML (what NETCONF sends). Use `native` to verify correctness, `xml` to debug YANG mapping issues.

**Verification:**

Review the dry-run output. If correct, `commit`. If wrong, `revert` and fix the service template.

### Task 8.2 — Write a custom L3VPN service package

**Question:** Create an NSO service package that provisions L3VPN across any set of PEs with a single command. The service takes: customer-name, VRF-name, RD, RT, list of PEs with their CE-facing interfaces and CE ASNs.

**Solution:**

1. Skeleton: `ncs-make-package --service-skeleton python-and-template l3vpn-service`
2. YANG model (`src/yang/l3vpn-service.yang`):
```yang
list l3vpn {
  key customer-name;
  leaf customer-name { type string; }
  leaf vrf-name { type string; }
  leaf rd { type string; }
  leaf rt { type string; }
  list pe {
    key name;
    leaf name { type leafref { path "/ncs:devices/ncs:device/ncs:name"; } }
    leaf ce-interface { type string; }
    leaf ce-asn { type uint32; }
  }
}
```
3. XML template (`templates/l3vpn-service-template.xml`) maps to IOS-XR VRF + BGP config.
4. Deploy: copy to `packages/`, `packages reload`, then:
```
services l3vpn CUST_A vrf-name CUST_A rd 65012:100 rt 65012:100 pe E-R1 ce-interface GigabitEthernet0/0/0/0 ce-asn 65012
commit dry-run outformat native
commit
```

**Verification:**

`show running-config vrf CUST_A` on E-R1 — config pushed by NSO matches expected VRF setup.

### Task 8.3 — Multi-device atomic rollback

**Question:** You pushed a wrong RT (65012:999 instead of 65012:100) to E-R1, G-R1, and Gar-R1 via NSO. All 3 PEs now have broken VPN connectivity. Roll back all 3 in one atomic operation.

**Solution:**

```
admin@ncs# rollback configuration
admin@ncs# commit dry-run outformat native
```

Review — confirms it will revert the RT on all 3 PEs. Then:

```
admin@ncs# commit
```

**Verification:**

`show running-config vrf CUST_A` on all 3 PEs — RT back to 65012:100. Customer A traffic restored.

---

## Section 9: Advanced Python Workflows

### Task 9.1 — Python VRF consistency audit

**Question:** Write a Python script that connects to all PEs via NETCONF, pulls VRF CUST_A config, and reports any RT mismatches across PEs.

**Solution:**

```python
from ncclient import manager
import xml.etree.ElementTree as ET

PES = {
    'E-R1': '1.1.1.1', 'E-R2': '2.2.2.2',
    'G-R1': '21.21.21.21', 'G-R2': '22.22.22.22',
    'Gar-R1': '11.11.11.11', 'Gar-R2': '12.12.12.12'
}

VRF_FILTER = '''
<vrfs xmlns="http://cisco.com/ns/yang/Cisco-IOS-XR-infra-rsi-cfg">
  <vrf>
    <vrf-name>CUST_A</vrf-name>
  </vrf>
</vrfs>
'''

results = {}
for name, ip in PES.items():
    try:
        with manager.connect(host=ip, port=830, username='admin',
                             password='admin', hostkey_verify=False) as m:
            reply = m.get_config(source='running', filter=('subtree', VRF_FILTER))
            root = ET.fromstring(reply.xml)
            # Parse RT values from XML
            rts = [elem.text for elem in root.iter() if 'route-target' in elem.tag.lower()]
            results[name] = rts if rts else ['VRF NOT FOUND']
    except Exception as e:
        results[name] = [f'CONNECTION FAILED: {e}']

# Compare
baseline = results.get('E-R1', [])
for pe, rts in results.items():
    status = 'MATCH' if rts == baseline else 'MISMATCH'
    print(f'{pe:6s} RTs: {rts}  [{status}]')
```

**Verification:**

Run the script. Output should show MATCH for all PEs with CUST_A, or flag which PE has wrong/missing RTs.

### Task 9.2 — Python SR prefix-SID uniqueness check

**Question:** Write a script that validates no two Garnet routers share the same prefix-SID index.

**Solution:**

```python
from ncclient import manager
import xml.etree.ElementTree as ET

GARNET = {
    'Gar-R1': '11.11.11.11', 'Gar-R2': '12.12.12.12', 'Gar-R3': '13.13.13.13',
    'Gar-R4': '14.14.14.14', 'Gar-R5': '15.15.15.15', 'Gar-R7': '17.17.17.17',
    'Gar-R6': '16.16.16.16'
}

sid_map = {}
for name, ip in GARNET.items():
    with manager.connect(host=ip, port=830, username='admin',
                         password='admin', hostkey_verify=False) as m:
        reply = m.get_config(source='running')
        root = ET.fromstring(reply.xml)
        for elem in root.iter():
            if 'prefix-sid' in elem.tag.lower() and 'index' in elem.tag.lower():
                idx = elem.text
                if idx in sid_map:
                    print(f'CONFLICT: {name} and {sid_map[idx]} both use index {idx}')
                else:
                    sid_map[idx] = name
                    print(f'{name:8s} prefix-sid index {idx} — OK')

print(f'\nTotal: {len(sid_map)} unique SIDs, 0 conflicts' if len(sid_map) == len(GARNET) else 'CONFLICTS DETECTED')
```

**Verification:**

Script reports all 7 Garnet routers with unique prefix-SID indexes. If you intentionally set two routers to the same index, the script catches it.

---

## Section 10: MDT Advanced (XPATH + Event-Driven)

### Task 10.1 — XPATH-filtered telemetry subscription

**Question:** Configure a telemetry subscription that streams ONLY BGP neighbor connection state changes — not the entire BGP operational tree.

**Solution:**

```
telemetry model-driven
 sensor-group BGP-STATE
  sensor-path Cisco-IOS-XR-ipv4-bgp-oper:bgp/instances/instance/instance-active/default-vrf/neighbors/neighbor/connection-state
 !
 subscription BGP-NEIGHBOR-WATCH
  sensor-group-id BGP-STATE sample-interval 10000
  destination-id COLLECTOR
 !
!
```

Explanation: Without XPATH filtering, streaming all of `Cisco-IOS-XR-ipv4-bgp-oper` sends thousands of counters per neighbor (prefixes, timers, message stats). The specific path narrows to just `connection-state` — one field per neighbor, every 10 seconds.

**Verification:**

On the collector, verify only `connection-state` updates arrive (Established/Idle), not full BGP table dumps. Shut a BGP neighbor — collector shows state change within 10 seconds.

### Task 10.2 — Event-driven vs cadence-based telemetry

**Question:** Configure an event-driven (on-change) subscription for IS-IS adjacency state, and a cadence-based (sample-interval) subscription for interface counters. Explain when to use each.

**Solution:**

```
telemetry model-driven
 sensor-group ISIS-ADJ
  sensor-path Cisco-IOS-XR-clns-isis-oper:isis/instances/instance/neighbors/neighbor
 !
 sensor-group INTF-COUNTERS
  sensor-path Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters
 !
 subscription ISIS-ONCHANGE
  sensor-group-id ISIS-ADJ sample-interval 0    ← 0 = on-change (event-driven)
  destination-id COLLECTOR
 !
 subscription INTF-POLL
  sensor-group-id INTF-COUNTERS sample-interval 30000  ← every 30 seconds
  destination-id COLLECTOR
 !
!
```

`sample-interval 0` = event-driven. The router only sends data when the IS-IS adjacency state changes (Up→Down or Down→Up). No periodic updates. Use for state-change events.

`sample-interval 30000` = cadence-based (30 sec). Counters update continuously — you need periodic snapshots to track trends. Use for metrics/counters.

**Verification:**

Shut an IS-IS interface — collector gets an immediate adjacency-down notification (not waiting for the next poll). Interface counters keep streaming every 30 seconds regardless.
