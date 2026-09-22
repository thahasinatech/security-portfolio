# Harden It — CIS Hardening of Ubuntu Server

## Status: complete

Baseline audit, Ansible hardening, and after-audit all done.
Hardening index moved from 58/100 to 69/100.

## What this is

Ubuntu Server from Lab Zero gets a security baseline audit, a set of
CIS-aligned hardening controls applied via Ansible, then a second audit to
see what actually moved. The point isn't just running a tool and calling it
done — it's a real before/after with the reasoning for what got fixed and
why.

## Baseline audit

Ubuntu Server was restored to its `clean-baseline-pre-hardening` snapshot
first, so the numbers reflect a clean, untouched install.

Lynis needed installing, which meant briefly adding a NAT adapter to Ubuntu
Server (Adapter 2) just for `apt install`, then removing it again once the
tools were in. The host-only isolation from Lab Zero was untouched — Adapter
1 stayed on the host-only network the whole time.

**Baseline hardening index: 58/100**, with 48 suggestions. The significant
ones:

- No firewall configured at all (`FIRE-4590`)
- SSH running mostly default — X11 forwarding on, agent forwarding on, no
  tightened auth-tries limit, TCP keepalive on, verbose logging off
- No password aging policy — no minimum age, no maximum age, no expiry dates
- One installed package flagged with a known vulnerability
- No PAM password-strength module installed
- Default umask (022) looser than it needs to be

## Applying the fixes — Ansible

Wrote a custom playbook (`ansible/harden.yml`) targeting the specific
findings above, rather than running a generic CIS role blind. Tasks:

- SSH hardening: disabled X11 forwarding, agent forwarding, TCP forwarding;
  lowered `MaxAuthTries` to 3 and `MaxSessions` to 2; enabled verbose
  logging; disabled TCP keepalive
- Enabled `ufw` with a default-deny inbound policy, explicitly allowing SSH
  so remote access wasn't cut off
- Set password aging policy (`PASS_MIN_DAYS`, `PASS_MAX_DAYS`) and a
  stricter default umask (027) in `/etc/login.defs`
- Installed `libpam-pwquality` for password strength enforcement
- Installed and enabled `fail2ban`

**Ran into two real issues applying this, both worth noting:**

1. Ansible's `become` (sudo escalation) hung and timed out when run with
   `connection: local` and an interactive password prompt (`-K`). Fixed by
   configuring passwordless sudo for the automation account
   (`/etc/sudoers.d/`) — the same pattern real CI/automation service
   accounts use, since there's no human present to type a password when a
   pipeline runs.
2. The `apt` tasks (installing `fail2ban` and `libpam-pwquality`) failed
   with DNS resolution errors even with the NAT adapter enabled — the
   second network interface came up without picking up a DNS server
   automatically. Fixed with `sudo dhclient <interface>` to force a DHCP
   lease, which pulled DNS config along with it.

Verified SSH access still worked (second session tested before closing the
first) before considering the hardening safe.

## After-audit

Re-ran the same Lynis scan post-hardening.

**After hardening index: 69/100** — an 11-point improvement.

**36 suggestions remaining**, down from 48 — 12 resolved. The reductions
line up with the areas the playbook targeted: SSH configuration, the
firewall gap, and password aging policy. The remaining 36 are mostly lower-
priority items the playbook didn't target on this pass (things like
partition layout, USB storage restrictions, and package-verification
tooling) — candidates for a future hardening pass rather than blockers.

## Files in this folder

Raw Lynis data:
- `lynis-baseline-report-before.dat`, `lynis-baseline-suggestions-before.txt`,
  `lynis-baseline.log` — baseline audit
- `lynis-after-report.dat`, `lynis-after-suggestions.txt`, `lynis-after.log`
  — after-hardening audit

Ansible:
- `ansible/harden.yml` — the actual playbook applied

Screenshots (`evidence/`):
- `ubuntu-lynis-score-before.png`, `ubuntu-lynis-suggestion-before.png` —
  baseline hardening index and suggestions list
- `ansible-start.png`, `ansible-playbook-final-run.png` — the Ansible run,
  start and successful completion
- `ufw-status.png`, `fail2ban-status.png`, `sshd-config-hardened.png` —
  proof each control actually applied
- `lynis-after-score.png`, `lynis-suggestion-count-comparison.png` — the
  after-hardening results

## Reproducing this

1. Restore Ubuntu Server to a clean snapshot
2. Run baseline: `sudo lynis audit system -Q`
3. Enable NAT temporarily, install Ansible + the `community.general`
   collection, then disable NAT again
4. Configure passwordless sudo for the automation account
5. Run `ansible-playbook harden.yml` from `ansible/`
6. Verify SSH access before closing your session
7. Re-run Lynis, compare before/after
