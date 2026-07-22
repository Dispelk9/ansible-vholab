# Correction: `snmp_discovery` does not run on Orb Agent restart

Status: **documented behavior, not a bug** - corrects a claim in `README.md`
that was true for `network_discovery` but doesn't hold for
`snmp_discovery`. Found 2026-07-21 while verifying the fix in
`docs/snmp-discovery-persistent-path-bug.md`.

## The incorrect assumption

`README.md`'s "Orb Agent discovery" section says restarting the container
re-runs "the policy" immediately. That's true for `network_discovery` -
confirmed by its own distinct log line, `"running scanner"`, appearing
within ~30s of every restart. It was reasonable to assume `snmp_discovery`
worked the same way.

It doesn't. This assumption cost real debugging time during the
persistent-path investigation: restarting `orb-agent` after fixing the
server-side SNMP bug appeared to do nothing, and it looked like the fix
hadn't worked - when actually the fix was fine, `snmp_discovery` just
hadn't been given a chance to run yet.

## What actually happens

Every restart, `snmp_discovery` logs the identical sequence regardless of
whether a real scan happens:
```
"received policies" -> "loaded device lookup extensions" -> "starting policy runner" -> "policy applied successfully"
```
`"policy applied successfully"` only means the policy was *loaded/scheduled* -
not that a probe occurred. The actual scan attempt has its own, different
log line: `"starting SNMP probe scan"`, followed by either
`"no hosts responded to SNMP probe"` or real results.

Grepping a full day of `orb-agent` logs (2026-07-21, `vhohetzner1`) for
`"starting SNMP probe scan"` found it exactly **once**, at `12:00:00.053Z` -
a clean boundary of `orb_snmp_schedule: "0 */6 * * *"` (00:00/06:00/12:00/
18:00 UTC). Five separate container restarts happened before and after that
timestamp (`06:31`, `12:05`, `12:25`, `14:31`, `14:47`); none of them
produced a second `"starting SNMP probe scan"` line. Conclusion:
`snmp_discovery` only actually probes on its own cron tick, full stop -
restarting the container reloads the policy but never triggers an
out-of-cycle scan.

(`network_discovery`'s `*/15 * * * *` schedule ticks often enough that its
restart-triggered runs are easy to conflate with its regular schedule -
worth being deliberate about which one you're actually looking at if you
ever need to re-check this.)

## Practical implications

- **Don't restart Orb Agent expecting an immediate SNMP re-scan** - it does
  nothing for that backend. It's still the right thing to do for
  `network_discovery` (nmap), and it does correctly reload `agent.yaml`
  changes so the *next* scheduled `snmp_discovery` tick uses them - it just
  won't run one early.
- **To force an immediate SNMP run** (e.g. to verify a fix without waiting
  up to 6 hours): temporarily tighten the schedule, apply, wait, then
  revert:
  ```yaml
  # group_vars/vhodockers.yml, temporarily
  orb_snmp_schedule: "*/2 * * * *"
  ```
  ```bash
  ansible-playbook -i inventory netbox_discovery_deploy.yml --tags orb_agent
  # wait ~2 minutes, check for a fresh "starting SNMP probe scan"
  # then revert orb_snmp_schedule back to "0 */6 * * *" and re-apply
  ```
  Don't leave it on a tight schedule permanently - device/interface
  inventory doesn't change often enough to justify polling every 2 minutes,
  and it's needless load for no benefit.
- **When diagnosing "SNMP discovery isn't producing anything"**, always
  check for the backend-specific `"starting SNMP probe scan"` line first,
  not just `"policy applied successfully"` - the latter proves nothing
  about whether a scan actually ran.
