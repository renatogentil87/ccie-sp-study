# AWS Networking Fundamentals — Week 2: Security Groups vs NACLs

## The Call That Started This Article

"The application worked yesterday. Nothing changed. Now the database connection times out."

The network team checked the route tables. Fine. The subnets. Fine. The internet gateway. Fine. Ping works. TCP handshake works on port 443. But the app can't reach the database on port 5432.

Someone tightened a security rule on Friday. They didn't realize there are **two** layers of filtering in AWS — and they only updated one.

---

## Two Firewalls, Not One

In an enterprise network, you have one firewall between zones. Traffic hits it once. Rules are evaluated. Allow or deny. Simple.

In AWS, every packet is evaluated **twice**:

1. **Network ACL (NACL)** — at the subnet boundary
2. **Security Group** — at the instance (ENI) level

Both can block traffic. Both are invisible to each other. A packet allowed by the NACL can still be dropped by the Security Group, and vice versa.

```
Enterprise:
  Server → Firewall → Server
  One evaluation. One rule set.

AWS:
  Instance → Security Group → NACL → Subnet → NACL → Security Group → Instance
  Four evaluations (SG out, NACL out, NACL in, SG in) for a single flow.
```

---

## The Enterprise Network Comparison

If you've managed an enterprise network, here's the mental model:

```
Enterprise                      AWS
──────────────────────────────────────────────
Stateful firewall (ASA, Palo)  → Security Group
  Tracks connections             Tracks connections
  Return traffic auto-allowed    Return traffic auto-allowed
  Per-server rules               Per-instance rules
  Applied at the interface       Applied at the ENI

Stateless ACL (switch/router)  → Network ACL (NACL)
  Every packet evaluated          Every packet evaluated
  No connection tracking          No connection tracking
  Must allow BOTH directions      Must allow BOTH directions
  Applied at the VLAN/subnet      Applied at the subnet boundary
```

The key insight: **Security Groups = your stateful firewall. NACLs = your switch ACLs.** You already know both concepts — they just have different names in AWS.

---

## Security Groups — Stateful (Your Firewall)

A Security Group is a **stateful firewall** attached to every ENI (Elastic Network Interface). It tracks connections.

```
Inbound rule: Allow TCP 5432 from 10.0.3.0/24
```

That's it. The return traffic (TCP responses from the database back to the app) is **automatically allowed** because the Security Group tracks the connection. You never write an outbound rule for return traffic.

**Key behaviors:**

```
1. STATEFUL — return traffic automatically allowed
2. ALLOW only — no explicit DENY rules (implicit deny everything not allowed)
3. Evaluated at the ENI — per-instance, not per-subnet
4. All rules evaluated — no order, no rule numbers (unlike ACLs)
5. Can reference other Security Groups — "allow from sg-webapp" instead of CIDRs
```

**Enterprise comparison:**

```
Cisco ASA:
  access-list INSIDE permit tcp 10.0.3.0 255.255.255.0 host 10.0.5.50 eq 5432
  → return traffic auto-allowed (stateful inspection)

AWS Security Group:
  Inbound: TCP 5432 from 10.0.3.0/24
  → return traffic auto-allowed (stateful)
  
  Same concept. Same behavior. Different syntax.
```

---

## Network ACLs — Stateless (Your Switch ACL)

A NACL is a **stateless packet filter** applied at the subnet boundary. It does NOT track connections. Every packet — inbound AND outbound — is evaluated independently.

```
Inbound rule #100: Allow TCP 5432 from 10.0.3.0/24
Outbound rule #100: Allow TCP source-port 5432 to 10.0.3.0/24   ← YOU MUST ADD THIS
```

Without the outbound rule, the database accepts the connection but the response packet is **dropped at the NACL** on the way out. The Security Group allowed it. The NACL blocked it. The app sees a timeout.

**Key behaviors:**

```
1. STATELESS — return traffic NOT auto-allowed (must write both directions)
2. ALLOW and DENY — explicit deny rules possible
3. Evaluated at the subnet — all instances in the subnet share the same NACL
4. Rules processed in ORDER — lowest number first, first match wins
5. Ephemeral ports — return traffic uses random high ports (1024-65535)
```

**Enterprise comparison:**

```
Cisco Router ACL:
  ip access-list extended INBOUND
    permit tcp 10.0.3.0 0.0.0.255 host 10.0.5.50 eq 5432
  ip access-list extended OUTBOUND
    permit tcp host 10.0.5.50 eq 5432 10.0.3.0 0.0.0.255
  → must allow BOTH directions (stateless)

AWS NACL:
  Inbound #100: Allow TCP 5432 from 10.0.3.0/24
  Outbound #100: Allow TCP 1024-65535 to 10.0.3.0/24     ← ephemeral ports
  → same concept, must allow both directions
```

---

## The Ephemeral Port Trap

This is where NACLs break applications silently.

When a client connects to a server on port 5432, the **return traffic** comes from port 5432 but goes to a **random ephemeral port** (1024-65535) on the client.

```
Client (10.0.3.50:49152) → Server (10.0.5.50:5432)    INBOUND to server subnet
Server (10.0.5.50:5432) → Client (10.0.3.50:49152)    OUTBOUND from server subnet
                                                        ↑ ephemeral port!
```

Your NACL outbound rule must allow the **ephemeral port range**, not just port 5432:

```
WRONG:
  Outbound #100: Allow TCP 5432 to 10.0.3.0/24        ← only allows source port 5432
  Result: response packets to port 49152 DROPPED

RIGHT:
  Outbound #100: Allow TCP 1024-65535 to 10.0.3.0/24  ← allows ephemeral return ports
  Result: response packets flow normally
```

Security Groups handle this automatically (stateful). NACLs don't (stateless). This is the #1 reason applications break when someone adds NACL rules without understanding stateless filtering.

---

## When to Use Which

```
Security Groups (always use):
  ✓ Primary security control for every instance
  ✓ Stateful — simple to manage
  ✓ Reference other SGs — "allow from the web tier SG"
  ✓ Per-instance granularity
  ✓ No risk of breaking return traffic

NACLs (use when needed):
  ✓ Subnet-level deny rules — block specific IPs/ranges
  ✓ Defense-in-depth — second layer behind SGs
  ✓ Compliance requirements — some standards require subnet-level filtering
  ✗ Easy to break things — must manage both directions
  ✗ Ephemeral ports add complexity
```

**Most AWS architectures use Security Groups as the primary control and leave NACLs at default (allow all).** NACLs are added only when you need explicit deny rules or compliance requires subnet-level filtering.

```
Enterprise equivalent:
  Security Groups = your primary firewall (ASA, Palo) — always on
  NACLs = switch ACLs — only when you need L3/L4 filtering at the network edge
```

---

## Side-by-Side Comparison

| Feature | Security Group | NACL |
|---------|---------------|------|
| **Stateful/Stateless** | Stateful | Stateless |
| **Return traffic** | Auto-allowed | Must explicitly allow |
| **Rule type** | Allow only | Allow AND Deny |
| **Rule order** | All rules evaluated | Processed in order (first match) |
| **Applied at** | ENI (instance) | Subnet boundary |
| **Scope** | Per-instance | All instances in subnet |
| **Default** | Deny all inbound, allow all outbound | Allow all (default NACL) |
| **SG references** | Yes ("allow from sg-xxxx") | No (CIDR only) |
| **Ephemeral ports** | Not needed (stateful) | Must allow 1024-65535 |
| **Cross-TGW** | References DON'T work | CIDR rules work |
| **Enterprise equivalent** | Stateful firewall (ASA/Palo) | Router/switch ACL |

---

## The Friday Incident — Explained

Back to the opening scenario. Someone "tightened security" by adding a NACL rule on the database subnet:

```
NACL Inbound #100: Allow TCP 5432 from 10.0.3.0/24      ← app subnet
NACL Inbound #200: Deny all                               ← block everything else
NACL Outbound: (left at default — allow all)              ← OK
```

This looks right. But the app still times out. Why?

The app connects from 10.0.3.50:**49152** to 10.0.5.50:**5432**. The inbound rule allows it. The database processes the request. The response goes from 10.0.5.50:**5432** to 10.0.3.50:**49152**.

Now that response hits the **app subnet's NACL** inbound:

```
App subnet NACL Inbound:
  #100: Allow TCP 443 from 0.0.0.0/0                     ← web traffic
  #200: Allow TCP 22 from 10.0.1.0/24                    ← SSH from bastion
  #999: Deny all                                          ← implicit deny
  
  Response packet: TCP from 10.0.5.50:5432 to 10.0.3.50:49152
  → matches #999 (deny all) → DROPPED
```

The response was allowed OUT of the database subnet but blocked INBOUND to the app subnet. The app subnet's NACL didn't have a rule allowing TCP from port 5432.

Fix:

```
App subnet NACL Inbound:
  #150: Allow TCP 1024-65535 from 10.0.5.0/24            ← database responses
```

Or simpler — remove the restrictive NACL rules and rely on Security Groups (which handle this automatically).

---

## Key Takeaways

1. **Security Groups are stateful. NACLs are stateless.** If you only remember one thing, remember this. Return traffic is the difference.

2. **Security Groups first. NACLs only when needed.** Don't add NACLs "for extra security" unless you understand ephemeral ports and bidirectional rules.

3. **SG references don't work across Transit Gateway.** If your app in VPC-A references `sg-webapp` but the database is in VPC-B via TGW — the reference doesn't resolve. Use CIDRs for cross-VPC rules.

4. **NACLs process in order. First match wins.** A deny at rule #50 overrides an allow at rule #100. Security Groups evaluate all rules — no ordering.

5. **When the app worked yesterday and nothing changed — check NACLs.** Someone on another team may have modified the subnet-level NACL without telling you.

---

*Next week: VPC Connectivity — Peering, Transit Gateway, and PrivateLink. When to use each and what happens when you choose wrong.*

*This is Week 2 of 7 in the AWS Networking Fundamentals series.*

---

**#AWS #VPC #SecurityGroups #NACL #Networking #CloudNetworking #Fundamentals #Week2**
