# SNMP discovery never worked: `/var/lib/snmpd` vs `/var/lib/snmp`

Status: **fixed** (`roles/snmp_agent/tasks/main.yml`), 2026-07-21. Both
vhohetzner1 and vhohetzner2 were affected from the moment this role was
first written - `snmp_discovery` had never once worked on either host.

## Symptom

- Orb Agent's `snmp_discovery` policy always logged `"policy applied
  successfully"` - no errors, looked healthy.
- Every manual `snmpget`/`snmpwalk` against either host, with the exact
  credentials from the vault, failed with `Unknown user name` - on
  `10.0.0.2`, on `10.0.0.3` (querying itself), always.
- NetBox only ever showed bare `IP Address` objects (from `network_discovery`
  / nmap) - never the `Device`/`Interface`/`Prefix` objects `snmp_discovery`
  is supposed to produce.
- `grep -i orb-discovery /var/lib/snmpd/snmpd.conf` always showed the
  **cleartext** `createUser orb-discovery SHA "..." AES "..."` line, no
  matter how many times `snmpd` was restarted, or what permissions were set
  on that file/directory.

## Root cause

**`roles/snmp_agent/tasks/main.yml` wrote the SNMPv3 `createUser` line to
`/var/lib/snmpd/snmpd.conf` - net-snmp's actual persistent-state directory
on this Debian/Ubuntu build is `/var/lib/snmp` (no trailing `d`).** One
letter, wrong the entire time. `snmpd` never opened, read, or processed
that file at all - the `createUser` line just sat there inert forever,
regardless of restarts or permissions, because nothing was ever looking at
it.

Confirmed directly from a live `snmpd -DALL` debug trace:
```
read_config:path:  config path used for snmpd:/var/lib/snmp (persistent path:/var/lib/snmp)
```

## Why it took so long to find (dead ends, in order - don't repeat these)

1. **Assumed the ACL (`rouser`) was too restrictive.** It wasn't - `rouser
   <user> priv` with no OID grants the whole tree by default; net-snmp
   confirms this.
2. **Assumed `snmpd` just needed a restart** to pick up the `createUser`
   line (net-snmp *does* localize+rewrite `createUser` lines on startup -
   this is real behavior, just for the file it's actually configured to
   read, which wasn't the one being edited).
3. **Assumed a file-permission problem** on `/var/lib/snmpd/snmpd.conf`
   (`root:root 0600`, unreadable by the `Debian-snmp` user `snmpd` actually
   runs as). This was *also* a real bug (see below) and looked exactly like
   the symptom, which is what made it such a convincing dead end - fixing it
   (even escalating to full `0770`/`0660` read+write) changed nothing,
   because it was the wrong directory the whole time.
4. **Suspected AppArmor/MAC confinement** next, since DAC permissions being
   "obviously correct" and still failing looked like a MAC layer blocking
   it. Never actually confirmed either way - the real answer was simpler.
5. **Checked NetBox Labs' own docs** for platform-support gaps (was `CX22`,
   a Hetzner VPS plan, simply unsupported by the `snmp_discovery` backend?)
   and a separate `device_discovery` (NAPALM/SSH) backend as a possible
   alternative. Both ruled out cleanly - `snmp_discovery` is vendor-agnostic
   at the protocol level, and `device_discovery` is for network vendor gear
   over SSH, not applicable here at all. Useful to rule out, but one layer
   above where the actual bug lived.
6. **A `diode-reconciler` `invalid_client` OAuth error** (Hydra
   client-registration startup race) surfaced in parallel during this same
   session and was a real, separate, self-healing issue - easy to
   conflate with the SNMP problem since both showed up around the same
   time. They were unrelated.
7. **Finally traced it directly**: ran `snmpd` in the foreground with full
   debug output (`snmpd -f -Le -D ...` - see below), captured it to a file,
   and grepped for `persistent` - which immediately surfaced the real path.

## The technique that actually worked

When a daemon's behavior doesn't match its config file and nothing in
`journalctl` explains why, don't keep guessing at permissions/config
content - **ask the daemon itself, in the foreground, with debug tracing
on**:

```bash
systemctl stop <service>
ss -ulnp | grep :<port>          # confirm the port is actually free first
<binary> -f -Le -D <exact same flags systemd uses> 2>&1 | tee /tmp/debug.log
# (get the exact flags from `systemctl cat <service> | grep ExecStart` -
# don't reconstruct them by hand from a `ps aux` snapshot, spacing/quoting
# differences there can produce a completely different, misleading crash)
```

Trigger the behavior you're debugging from another terminal while it runs,
then `Ctrl-C` and grep the **file**, not the scrollback - a bare `-D` (all
debug tokens) firehoses far more output than a terminal can usefully show
live, especially on a host doing anything else in parallel (Docker's
`veth*` interface churn, in this case).

One more trap worth naming: net-snmp's **persisted** `usmUser` line stores
the security name as a **hex-encoded octet string**
(`0x6f72622d646973636f76657279` = `"orb-discovery"`), never as literal
quoted text. A plain `grep -i <username>` against the persistent file will
report nothing even on a fully successful, correctly localized user - this
looked like another failure and wasn't. Check for a bare `^usmUser ` line
instead if you just need to confirm *a* user was provisioned.

## The fix

`roles/snmp_agent/tasks/main.yml`:
- Both the idempotency check and the `createUser` provisioning task now
  target `/var/lib/snmp/snmpd.conf` (not `/var/lib/snmpd/snmpd.conf`).
- No `owner`/`group`/`mode` is set on that file by the role at all -
  `/var/lib/snmp` is created and permissioned correctly by the `snmpd`
  package itself; managing ownership there risks re-breaking access rather
  than fixing anything (this is what the failed permission escalation in
  dead-end #3 above taught us).
- The idempotency check matches a bare `^usmUser ` line rather than the
  literal username, since the persisted form hex-encodes the security name
  (see above).

The stray `/var/lib/snmpd` directory (the wrong path) may still exist on
both hosts with leftover permission changes from live debugging
(`root:Debian-snmp`, `0770`/`0660`) - harmless dead weight, safe to leave or
manually clean up, nothing reads it.

## Verifying this is actually fixed

```bash
ansible-playbook -i inventory update_hetzner.yml
grep -i orb-discovery /var/lib/snmp/snmpd.conf   # should show a usmUser line with hex-encoded key material
snmpget -v3 -u orb-discovery -l authPriv -a SHA -A '<auth_passphrase>' -x AES -X '<priv_passphrase>' udp:10.0.0.2:161 1.3.6.1.2.1.1.1.0
```
Should return a real `sysDescr` string, not `Unknown user name`. Then
restart Orb Agent (`cd /opt/netbox/discovery/orb && docker compose restart
orb-agent`) and check NetBox's DCIM → Devices/Interfaces for the
SNMP-derived objects reappearing.
