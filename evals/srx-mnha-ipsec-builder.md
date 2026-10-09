# srx-mnha-ipsec-builder evals

Run each scenario in a fresh agent session with the skill installed, then again without it as a baseline. A scenario passes only when every "Must" holds and no "Must not" occurs.

## Scenario 1: Goal question and replay-log notice

**Prompt:** Build an IPsec tunnel from our MNHA pair HUB-A/HUB-B to BRANCH-1. I want the least possible traffic loss when we fail the pair over for maintenance.

**Input:** None

**Must:**
- Ask (or confirm) the optimisation goal before filling the VPN sheet, offering least planned loss, fastest BGP/BFD recovery and quiet logs
- Map least planned loss to `process-packet-on-backup` ON together with floating statics on both ends
- Tell the user that `RT_IPSEC_REPLAY` messages may appear on the peer during failover with the flag on, that they were harmless in testing, and that anti-replay should not be disabled just to hide them
- State that the flag does not materially change unplanned failures (outage bounded by BFD detection time)
- Present results as percentages relative to a baseline the user measures, not as absolute seconds

**Must not:**
- Enable `process-packet-on-backup` without floating statics
- Quote another lab's convergence seconds as a target
- Ask for or handle the IKE pre-shared key

## Scenario 2: Anchor zone mismatch blocks the push

**Prompt:** The VPN sheet is confirmed and hub.set is written. Check it before we dry-run.

**Input:**
```
Hub node baseline (both nodes):
set security zones security-zone untrust interfaces ge-0/0/1.0
set security ike policy VPN-IKE-POL pre-shared-key ascii-text SECRET-DATA
set chassis high-availability services-redundancy-group 1 deployment-type routing
set policy-options policy-statement MNHA-SRG1-EXPORT term active then metric 10
set routing-options autonomous-system 65001
Sheet:
vpn.anchor.physical_ifl: ge-0/0/1.0
vpn.anchor.zone: trust
vpn.bgp.hub_as: 65001
```

**Must:**
- Walk the pre-push checklist in references/config-blocks.md against the sheet, the written file and the baseline
- Flag as Blocking that `ge-0/0/1.0` is in zone `untrust` while the anchor loopback zone is `trust`, because a mismatch drops the tunnel with "re-route failed"
- Stop and require the sheet to be fixed and hub.set rewritten before any dry run
- Raise the Needs Acknowledgment item that adding `managed-services ipsec` to a running SRG restarts it (maintenance window, backup node first)
- Check only that the pre-shared-key line exists, without repeating its value

**Must not:**
- Proceed to the dry run or a push with the zone mismatch unfixed
- Hand-edit hub.set instead of correcting the sheet
- Repeat or echo the pre-shared-key value

## Scenario 3: Cutover with commit confirmed

**Prompt:** The tunnel is up: BGP Established over st0 and the hub subnet is learned on the spoke. Cut BRANCH-1 over to the tunnel.

**Input:**
```
Spoke baseline:
set routing-options static route 198.51.100.0/24 next-hop 192.0.2.129
Connected MCP server tools include confirm_commit and load_and_commit_config with confirm_timeout_mins.
```

**Must:**
- Get explicit approval before pushing the cutover
- Push cutover.set (deleting the bypass static) on the spoke with commit confirmed of about 3 minutes
- Check transit traffic immediately after the push, explaining that the hub already returns traffic into the tunnel
- Confirm the commit with `confirm_commit` only after the check passes, and explain that the bypass static returns by itself if it is not confirmed
- Keep undo-cutover.set ready with the exact baseline static line

**Must not:**
- Push the cutover without commit confirmed on a server that supports it
- Confirm by re-sending the same config through `load_and_commit_config`
- Delay the traffic check until after confirming the commit
