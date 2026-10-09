# MNHA IPsec Configuration Block Reference

## Contents

- [Introduction](#introduction)
- [Goal resolution](#goal-resolution)
- [Placeholder mapping](#placeholder-mapping)
- [Hub block (both MNHA nodes)](#hub-block-both-mnha-nodes)
- [Spoke block](#spoke-block)
- [Cutover files (spoke)](#cutover-files-spoke)
- [Undo files](#undo-files)
- [Pre-push checklist](#pre-push-checklist)

## Introduction

After the VPN sheet is confirmed, write each file by substituting `<PLACEHOLDER>` values
from the sheet into the blocks below. A written file must contain no placeholders and
no template syntax when it is pushed. Lines marked *(repeat per ...)* are written once
for each list item; lines marked *(only if ...)* are written only when the condition
holds. Drop blank lines.

The hub file is **identical on both MNHA nodes**: write it once, push it to the backup
node first.

## Goal resolution

The sheet's `vpn.goal` sets three switches. Optional sheet overrides
(`process_packet_on_backup`, `floating_statics`, `spoke_anti_replay`) win over the goal.

| `vpn.goal` | `<PPOB>` (process-packet-on-backup) | `<STATICS>` (floating statics) | `<ANTI_REPLAY>` (spoke) |
|---|---|---|---|
| `least-planned-loss` | on | on | on |
| `fastest-bgp-recovery` | off | on | on |
| `quiet-logs` | on | on | **off** (`no-anti-replay`) |

## Placeholder mapping

| Placeholder | Sheet field | Notes |
|---|---|---|
| `<SRG>` | `vpn.srg` | SRG1 or higher, never 0 |
| `<IKE_POLICY>` | `vpn.ike.policy` | Set up by the user with the PSK on every device |
| `<DPD_INTERVAL>` / `<DPD_THRESHOLD>` | `vpn.ike.dpd.interval` / `.threshold` | Default 3 / 3 |
| `<PFS_GROUP>` | `vpn.ipsec.pfs_group` | e.g. `group20` |
| `<IPSEC_LIFETIME>` | `vpn.ipsec.lifetime` | Seconds, e.g. `3600` |
| `<ANCHOR_IP>` | `vpn.anchor.ip` | Floating IKE address, no mask |
| `<LO0_UNIT>` | `vpn.anchor.lo0_unit` | Same unit on both hub nodes |
| `<ANCHOR_ZONE>` | `vpn.anchor.zone` | Zone of the physical interface that receives IKE/ESP |
| `<PHYSICAL_IFL>` | `vpn.anchor.physical_ifl` | e.g. `ge-0/0/1.0` |
| `<ST0_UNIT>` | `vpn.tunnel.st0_unit` | Same unit on hub and spoke |
| `<HUB_TUNNEL_IP>` / `<SPOKE_TUNNEL_IP>` | `vpn.tunnel.hub_ip` / `.spoke_ip` | Tunnel addresses |
| `<TUNNEL_PREFIX>` | `vpn.tunnel.prefix_len` | e.g. `30` |
| `<VPN_ZONE>` | `vpn.tunnel.zone` | Zone of `st0.<ST0_UNIT>` on both ends |
| `<HUB_AS>` / `<SPOKE_AS>` | `vpn.bgp.hub_as` / `.spoke_as` | eBGP only: must differ |
| `<BFD_MIN>` / `<BFD_MULT>` | `vpn.bgp.bfd.min_interval` / `.multiplier` | Default 300 / 3 |
| `<EXPORT_POLICY>` | `vpn.hub_export_policy` | The pair's **existing** MNHA export policy |
| `<HUB_SUBNET>` | `vpn.subnets.hub[]` | Hub LANs advertised to the spoke |
| `<SPOKE_SUBNET>` | `vpn.subnets.spoke[]` | Spoke LANs advertised to the hub |
| `<SPOKE_NAME>` | `spoke.router_name` | Used in default object names |
| `<SPOKE_IKE_IP>` | `spoke.ike_ip` | Spoke's IKE source address |
| `<SPOKE_EXT_IFL>` / `<SPOKE_EXT_ZONE>` | `spoke.external_ifl` / `.external_zone` | Spoke's IKE-facing interface and zone |
| `<UNDERLAY_NEXTHOP>` | `spoke.underlay_nexthop` | Spoke's next hop toward the anchor |
| `<BYPASS_PREFIX>` | `spoke.bypass_static[]` | Pre-tunnel statics to the hub subnets |
| `<ADDR_NAME>` / `<ADDR_PREFIX>` | `address_book.hub[]` / `.spoke[]` | Global address-book entries |
| `<POL_*>` | `policies.hub[]` / `.spoke[]` | `name`, `from_zone`, `to_zone`, `source`, `destination`, `application`, `log` |

Default object names (override them in an optional `names:` block of the sheet):

| Placeholder | Default |
|---|---|
| `<HUB_GW>` / `<HUB_VPN>` | `VPN-SPOKE-<SPOKE_NAME>` / `VPN-<SPOKE_NAME>` |
| `<SPOKE_GW>` / `<SPOKE_VPN>` | `VPN-HUB-MNHA` / `VPN-HUB` |
| `<IPSEC_PROP>` / `<IPSEC_POL>` | `VPN-IPSEC-PROP` / `VPN-IPSEC-POL` |
| `<ANCHOR_PREFIX_LIST>` | `MNHA-IPSEC-ANCHOR` |
| `<HUB_IN_POLICY>` / `<HUB_OUT_POLICY>` / `<HUB_BGP_GROUP>` | `VPN-SPOKES-IN` / `VPN-SPOKES-OUT` / `VPN-SPOKES` |
| `<SPOKE_IN_POLICY>` / `<SPOKE_OUT_POLICY>` / `<SPOKE_BGP_GROUP>` | `VPN-HUB-IN` / `VPN-HUB-OUT` / `VPN-HUB` |

## Hub block (both MNHA nodes)

Anchor, tunnel interface and SRG:

```junos
set interfaces lo0 unit <LO0_UNIT> family inet address <ANCHOR_IP>/32
set interfaces st0 unit <ST0_UNIT> family inet address <HUB_TUNNEL_IP>/<TUNNEL_PREFIX>
set policy-options prefix-list <ANCHOR_PREFIX_LIST> <ANCHOR_IP>/32
set chassis high-availability services-redundancy-group <SRG> managed-services ipsec
set chassis high-availability services-redundancy-group <SRG> prefix-list <ANCHOR_PREFIX_LIST>
```

*(only if `<PPOB>` is on)*

```junos
set chassis high-availability services-redundancy-group <SRG> process-packet-on-backup
```

Zones, IKE and IPsec:

```junos
set security zones security-zone <ANCHOR_ZONE> interfaces lo0.<LO0_UNIT>
set security zones security-zone <ANCHOR_ZONE> host-inbound-traffic system-services ike
set security zones security-zone <VPN_ZONE> interfaces st0.<ST0_UNIT>
set security zones security-zone <VPN_ZONE> host-inbound-traffic system-services ping
set security zones security-zone <VPN_ZONE> host-inbound-traffic protocols bgp
set security zones security-zone <VPN_ZONE> host-inbound-traffic protocols bfd
set security ike gateway <HUB_GW> ike-policy <IKE_POLICY>
set security ike gateway <HUB_GW> address <SPOKE_IKE_IP>
set security ike gateway <HUB_GW> dead-peer-detection probe-idle-tunnel
set security ike gateway <HUB_GW> dead-peer-detection interval <DPD_INTERVAL>
set security ike gateway <HUB_GW> dead-peer-detection threshold <DPD_THRESHOLD>
set security ike gateway <HUB_GW> local-address <ANCHOR_IP>
set security ike gateway <HUB_GW> external-interface lo0.<LO0_UNIT>
set security ike gateway <HUB_GW> version v2-only
set security ipsec proposal <IPSEC_PROP> protocol esp
set security ipsec proposal <IPSEC_PROP> encryption-algorithm aes-256-gcm
set security ipsec proposal <IPSEC_PROP> lifetime-seconds <IPSEC_LIFETIME>
set security ipsec policy <IPSEC_POL> perfect-forward-secrecy keys <PFS_GROUP>
set security ipsec policy <IPSEC_POL> proposals <IPSEC_PROP>
set security ipsec vpn <HUB_VPN> bind-interface st0.<ST0_UNIT>
set security ipsec vpn <HUB_VPN> ike gateway <HUB_GW>
set security ipsec vpn <HUB_VPN> ike ipsec-policy <IPSEC_POL>
set security ipsec vpn <HUB_VPN> establish-tunnels immediately
```

Intra-zone IKE and ESP policy. **Always written, required even if its hit count stays
0** (`mnha-ipsec.md`):

```junos
set applications application VPN-ESP protocol esp
set security policies from-zone <ANCHOR_ZONE> to-zone <ANCHOR_ZONE> policy ALLOW-IKE-ESP match source-address any
set security policies from-zone <ANCHOR_ZONE> to-zone <ANCHOR_ZONE> policy ALLOW-IKE-ESP match destination-address any
set security policies from-zone <ANCHOR_ZONE> to-zone <ANCHOR_ZONE> policy ALLOW-IKE-ESP match application junos-ike
set security policies from-zone <ANCHOR_ZONE> to-zone <ANCHOR_ZONE> policy ALLOW-IKE-ESP match application junos-ike-nat
set security policies from-zone <ANCHOR_ZONE> to-zone <ANCHOR_ZONE> policy ALLOW-IKE-ESP match application VPN-ESP
set security policies from-zone <ANCHOR_ZONE> to-zone <ANCHOR_ZONE> policy ALLOW-IKE-ESP then permit
```

Scope `source-address` and `destination-address` to the peer and the anchor where
possible.

Address book and transit policies *(repeat per `address_book.hub[]` entry, then per
`policies.hub[]` entry)*:

```junos
set security address-book global address <ADDR_NAME> <ADDR_PREFIX>
set security policies from-zone <POL_FROM> to-zone <POL_TO> policy <POL_NAME> match source-address <POL_SOURCE>
set security policies from-zone <POL_FROM> to-zone <POL_TO> policy <POL_NAME> match destination-address <POL_DESTINATION>
set security policies from-zone <POL_FROM> to-zone <POL_TO> policy <POL_NAME> match application <POL_APPLICATION>
set security policies from-zone <POL_FROM> to-zone <POL_TO> policy <POL_NAME> then permit
```

*(only if the policy's `log` is true)*

```junos
set security policies from-zone <POL_FROM> to-zone <POL_TO> policy <POL_NAME> then log session-init
set security policies from-zone <POL_FROM> to-zone <POL_TO> policy <POL_NAME> then log session-close
```

Advertise the anchor upstream with the pair's existing active/backup steering, then the
tunnel BGP policies *(the `route-filter` lines repeat per `<SPOKE_SUBNET>` or
`<HUB_SUBNET>`)*:

```junos
set policy-options policy-statement <EXPORT_POLICY> term active from route-filter <ANCHOR_IP>/32 exact
set policy-options policy-statement <EXPORT_POLICY> term backup from route-filter <ANCHOR_IP>/32 exact
set policy-options policy-statement <HUB_IN_POLICY> term spoke-lan from route-filter <SPOKE_SUBNET> exact
set policy-options policy-statement <HUB_IN_POLICY> term spoke-lan then local-preference 200
set policy-options policy-statement <HUB_IN_POLICY> term spoke-lan then accept
set policy-options policy-statement <HUB_IN_POLICY> term default then reject
set policy-options policy-statement <HUB_OUT_POLICY> term hub-lan from route-filter <HUB_SUBNET> exact
set policy-options policy-statement <HUB_OUT_POLICY> term hub-lan then accept
set policy-options policy-statement <HUB_OUT_POLICY> term default then reject
set protocols bgp group <HUB_BGP_GROUP> type external
set protocols bgp group <HUB_BGP_GROUP> local-address <HUB_TUNNEL_IP>
set protocols bgp group <HUB_BGP_GROUP> peer-as <SPOKE_AS>
set protocols bgp group <HUB_BGP_GROUP> neighbor <SPOKE_TUNNEL_IP>
set protocols bgp group <HUB_BGP_GROUP> bfd-liveness-detection minimum-interval <BFD_MIN>
set protocols bgp group <HUB_BGP_GROUP> bfd-liveness-detection multiplier <BFD_MULT>
set protocols bgp group <HUB_BGP_GROUP> graceful-restart
set protocols bgp group <HUB_BGP_GROUP> import <HUB_IN_POLICY>
set protocols bgp group <HUB_BGP_GROUP> export <HUB_OUT_POLICY>
```

Floating statics carry traffic while the tunnel BGP re-converges. **Mandatory when
`<PPOB>` is on.** *(only if `<STATICS>` is on; repeat per `<SPOKE_SUBNET>`)*. No BFD on
these statics: BFD is node-local and would flap with the failover.

```junos
set routing-options static route <SPOKE_SUBNET> next-hop st0.<ST0_UNIT>
set routing-options static route <SPOKE_SUBNET> preference 250
```

## Spoke block

```junos
set interfaces st0 unit <ST0_UNIT> family inet address <SPOKE_TUNNEL_IP>/<TUNNEL_PREFIX>
set security zones security-zone <SPOKE_EXT_ZONE> host-inbound-traffic system-services ike
set security zones security-zone <VPN_ZONE> interfaces st0.<ST0_UNIT>
set security zones security-zone <VPN_ZONE> host-inbound-traffic system-services ping
set security zones security-zone <VPN_ZONE> host-inbound-traffic protocols bgp
set security zones security-zone <VPN_ZONE> host-inbound-traffic protocols bfd
set security ike gateway <SPOKE_GW> ike-policy <IKE_POLICY>
set security ike gateway <SPOKE_GW> address <ANCHOR_IP>
set security ike gateway <SPOKE_GW> dead-peer-detection probe-idle-tunnel
set security ike gateway <SPOKE_GW> dead-peer-detection interval <DPD_INTERVAL>
set security ike gateway <SPOKE_GW> dead-peer-detection threshold <DPD_THRESHOLD>
set security ike gateway <SPOKE_GW> local-address <SPOKE_IKE_IP>
set security ike gateway <SPOKE_GW> external-interface <SPOKE_EXT_IFL>
set security ike gateway <SPOKE_GW> version v2-only
set security ipsec proposal <IPSEC_PROP> protocol esp
set security ipsec proposal <IPSEC_PROP> encryption-algorithm aes-256-gcm
set security ipsec proposal <IPSEC_PROP> lifetime-seconds <IPSEC_LIFETIME>
set security ipsec policy <IPSEC_POL> perfect-forward-secrecy keys <PFS_GROUP>
set security ipsec policy <IPSEC_POL> proposals <IPSEC_PROP>
set security ipsec vpn <SPOKE_VPN> bind-interface st0.<ST0_UNIT>
set security ipsec vpn <SPOKE_VPN> ike gateway <SPOKE_GW>
set security ipsec vpn <SPOKE_VPN> ike ipsec-policy <IPSEC_POL>
set security ipsec vpn <SPOKE_VPN> establish-tunnels immediately
```

*(only if `<ANTI_REPLAY>` is off, goal `quiet-logs`)*: applies to **new** SAs only, so
clear the SAs on the spoke after committing and confirm a fresh SA.

```junos
set security ipsec vpn <SPOKE_VPN> ike no-anti-replay
```

Underlay route to the anchor and tunnel BGP *(the `route-filter` lines repeat per
`<HUB_SUBNET>` or `<SPOKE_SUBNET>`)*:

```junos
set routing-options static route <ANCHOR_IP>/32 next-hop <UNDERLAY_NEXTHOP>
set routing-options autonomous-system <SPOKE_AS>
set policy-options policy-statement <SPOKE_IN_POLICY> term hub-lan from route-filter <HUB_SUBNET> exact
set policy-options policy-statement <SPOKE_IN_POLICY> term hub-lan then accept
set policy-options policy-statement <SPOKE_IN_POLICY> term default then reject
set policy-options policy-statement <SPOKE_OUT_POLICY> term spoke-lan from protocol direct
set policy-options policy-statement <SPOKE_OUT_POLICY> term spoke-lan from route-filter <SPOKE_SUBNET> exact
set policy-options policy-statement <SPOKE_OUT_POLICY> term spoke-lan then accept
set policy-options policy-statement <SPOKE_OUT_POLICY> term default then reject
set protocols bgp group <SPOKE_BGP_GROUP> type external
set protocols bgp group <SPOKE_BGP_GROUP> local-address <SPOKE_TUNNEL_IP>
set protocols bgp group <SPOKE_BGP_GROUP> peer-as <HUB_AS>
set protocols bgp group <SPOKE_BGP_GROUP> neighbor <HUB_TUNNEL_IP>
set protocols bgp group <SPOKE_BGP_GROUP> bfd-liveness-detection minimum-interval <BFD_MIN>
set protocols bgp group <SPOKE_BGP_GROUP> bfd-liveness-detection multiplier <BFD_MULT>
set protocols bgp group <SPOKE_BGP_GROUP> graceful-restart
set protocols bgp group <SPOKE_BGP_GROUP> import <SPOKE_IN_POLICY>
set protocols bgp group <SPOKE_BGP_GROUP> export <SPOKE_OUT_POLICY>
```

Address book and transit policies: same block as the hub, using `address_book.spoke[]`
and `policies.spoke[]`.

Floating statics *(only if `<STATICS>` is on; repeat per `<HUB_SUBNET>`)*:

```junos
set routing-options static route <HUB_SUBNET> next-hop st0.<ST0_UNIT>
set routing-options static route <HUB_SUBNET> preference 250
```

## Cutover files (spoke)

`cutover.set` *(repeat per `<BYPASS_PREFIX>`)*:

```junos
delete routing-options static route <BYPASS_PREFIX>
```

`undo-cutover.set`: copy each removed bypass route's **exact** `set routing-options
static route <BYPASS_PREFIX> ...` lines from the spoke baseline, so the undo restores
the original next hop and any other attributes.

## Undo files

An undo file deletes only what its file **adds** relative to that device's baseline.
Configuration a file merely re-states (an existing zone, the existing export policy) is
never removed.

For each line of the file, working from the last line to the first:

1. Skip the line if it is already in the baseline.
2. Take the hierarchy path after `set` and find the shortest prefix of it that does not
   exist in the baseline.
3. If that prefix ends on a container keyword that names a list (`gateway`, `proposal`,
   `policy`, `vpn`, `security-zone`, `group`, `neighbor`, `term`, `policy-statement`,
   `unit`, `route`, `from-zone`, `to-zone`, `address`, `prefix-list`, `application`,
   `services-redundancy-group`, ...), extend it by one word so the delete names the
   object, never the whole list.
4. **`route-filter` lines are the exception:** delete the exact line
   (`delete policy-options policy-statement <EXPORT_POLICY> term active from route-filter <ANCHOR_IP>/32 exact`),
   never the term or the policy. This keeps the pair's existing export policy intact.
5. Write `delete <prefix>`, unless an earlier delete already covers it.

Example: if the baseline has `security-zone untrust` but no `lo0 unit 10`, the hub's
`set security zones security-zone untrust interfaces lo0.10` undoes as
`delete security zones security-zone untrust interfaces lo0.10`, and
`set interfaces lo0 unit 10 family inet address 192.0.2.1/32` undoes as
`delete interfaces lo0 unit 10`.

Undo files are valid only against the baseline they were computed from: dry-run each one
before relying on it.

## Pre-push checklist

Walk this after writing all files. Any **Blocking** item means stop: fix the sheet and
rewrite. **Needs Acknowledgment** items need a one-line user OK. **Tell the User** items
are context.

### Blocking

Sheet:
- [ ] No `<FILL>` remains in the sheet.
- [ ] `vpn.goal` is `least-planned-loss`, `fastest-bgp-recovery` or `quiet-logs`.
- [ ] `<PPOB>` is not on without `<STATICS>`: with the flag on, tunnel BFD/BGP recovers
      more slowly (about +70 % in testing) and nothing carries traffic meanwhile.
- [ ] Both tunnel IPs are in one subnet (`<TUNNEL_PREFIX>`) and are not identical.
- [ ] The anchor IP is not inside the tunnel subnet and is not the spoke's IKE IP.
- [ ] `hub.nodes` lists two different MNHA nodes.
- [ ] `<SPOKE_AS>` differs from `<HUB_AS>` (eBGP only).
- [ ] `<SRG>` is 1 or higher.

Written files:
- [ ] No line touches `interfaces fxp0`, `system services`, `system login`,
      `system root-authentication` or `routing-instances mgmt_junos`.
- [ ] No line contains `host-inbound-traffic system-services all`,
      `host-inbound-traffic protocols all` or `default-policy permit-all`.
- [ ] The hub file contains the `ALLOW-IKE-ESP` policy.
- [ ] No placeholder remains in any file.

Each hub node's baseline:
- [ ] `set chassis high-availability ...` is present (this skill needs a formed pair).
- [ ] A `set security ike policy <IKE_POLICY> pre-shared-key ...` line is present. Check
      only that the line exists; never read or repeat its value. The user sets it, then
      the baseline is re-taken.
- [ ] If `<PHYSICAL_IFL>` is in a zone, that zone is `<ANCHOR_ZONE>`. A mismatch drops
      the tunnel with "re-route failed".
- [ ] `lo0 unit <LO0_UNIT>` and `st0 unit <ST0_UNIT>` are not in use.
- [ ] IKE gateway `<HUB_GW>` does not exist yet.
- [ ] `<EXPORT_POLICY>` exists with a `term active` (build the pair's eBGP stage first).
- [ ] `routing-options autonomous-system` equals `<HUB_AS>`.

Spoke baseline:
- [ ] The `<IKE_POLICY>` pre-shared-key line is present (same rule as the hub).
- [ ] If `<SPOKE_EXT_IFL>` is in a zone, that zone is `<SPOKE_EXT_ZONE>`.
- [ ] `st0 unit <ST0_UNIT>` is not in use; IKE gateway `<SPOKE_GW>` does not exist yet.
- [ ] If `routing-options autonomous-system` is set, it equals `<SPOKE_AS>`.

### Needs Acknowledgment

- [ ] `<PPOB>` is on: `RT_IPSEC_REPLAY` messages may appear on the peer during failover
      (and occasionally at SA rekey). Measured as harmless; do not disable anti-replay
      just to silence them.
- [ ] `<ANTI_REPLAY>` is off (`quiet-logs`): removes replay protection on this tunnel;
      clear the SAs after committing so it applies.
- [ ] `spoke.bypass_static` is empty: if the spoke already reaches the hub subnets
      another way, the tunnel route will not take over.
- [ ] A listed bypass prefix is not in the spoke baseline.
- [ ] `<PHYSICAL_IFL>` is not in any zone in a hub baseline.
- [ ] `<PHYSICAL_IFL>` is in a non-default routing instance: the ICL, the anchor and
      that interface must share **one** instance (`mnha-ipsec.md`).
- [ ] No hub baseline has `managed-services ipsec` yet: adding it to a **running** SRG
      restarts it (the VIP was withdrawn for minutes on the active node in testing).
      Maintenance window, backup node first, re-check SRG state between nodes.
- [ ] A baseline has `default-policy permit-all`.
- [ ] A baseline has `deactivate` lines: confirm each is meant to stay inactive.
- [ ] A baseline was not supplied: its checks were skipped and its undo file deletes
      every written line.

### Tell the User

- [ ] `managed-services ipsec` is already set on an SRG of the pair.
- [ ] Each bypass static present in the spoke baseline is removed only by the cutover
      (Phase B, commit confirmed).
- [ ] Confirm an IKE process is running on the spoke (`show system processes` lists
      `iked` or `kmd`). `junos-ike` is required on the MNHA hub and recommended on the
      spoke; after installing it the node must reboot.
