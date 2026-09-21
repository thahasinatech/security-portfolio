# Harden It — CIS Hardening of Ubuntu Server

## Status: in progress

Baseline audit complete. Hardening and the after-audit come next.

## What this is

Ubuntu Server from Lab Zero gets a security baseline audit, then a set of
CIS-aligned hardening controls applied by hand (and later via Ansible), then
a second audit to see what actually moved. The point isn't just running a
tool and calling it done — it's showing a real before/after with the
reasoning for what got fixed and why.

## Baseline audit

Ubuntu Server was restored to its `clean-baseline-pre-hardening` snapshot
first, so the numbers below reflect a clean, untouched install — not
whatever state the VM happened to be in.

Lynis needed installing, which meant briefly adding a NAT adapter to Ubuntu
Server (Adapter 2) just for `apt install`, then removing it again once the
tools were in. The host-only isolation from Lab Zero was untouched the whole
time — Adapter 1 stayed on the host-only network throughout.

Ran the audit with:

```bash
sudo lynis audit system -Q
```

**Baseline hardening index: 58/100**

**48 suggestions** came back. A handful of the more significant ones:

- No firewall configured at all (`FIRE-4590`)
- SSH running with mostly default settings — X11 forwarding on, agent
  forwarding on, no limit on auth tries beyond the default 6, TCP keepalive
  on, verbose logging off
- No password aging policy — no minimum age, no maximum age, no expiry dates
- One installed package flagged with a known vulnerability
- No PAM password-strength module installed
- Default umask (022) is looser than it needs to be

Full output is in `evidence/`:
- `lynis-baseline-report.dat` — raw Lynis report data
- `lynis-baseline.log` — full audit log
- `lynis-baseline-suggestions.txt` — all 48 suggestions, extracted
- `ubuntu-lynis-score.png` — terminal screenshot of the hardening index
- `ubuntu-lynis-suggestion.png` — terminal screenshot of the suggestions list

## What's next

Applying a focused set of CIS-aligned controls rather than every one of the
48 suggestions — the ones that matter most for a real target box:

- SSH hardening (disable root login if not already, disable X11/agent
  forwarding, lower MaxAuthTries, enable verbose logging)
- Enable and configure a firewall (ufw) with a default-deny inbound policy
- Password aging policy (min/max age, expiry) in `/etc/login.defs`
- Install a PAM password-strength module
- Tighten the default umask
- Patch or remove the package flagged as vulnerable

After that: re-run Lynis, compare the hardening index and suggestion count
against this baseline, and write up what changed and why.
