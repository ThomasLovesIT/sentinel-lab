# Firewall Rules — FW01 (OPNsense)

## Status: In progress

## Interfaces configured
- WAN (em0) — NAT, DHCP
- LAN (em1) — 192.168.56.10/24, MGMT access
- OPT1 / SERVERS (em2) — 10.10.20.1/24
- OPT2 / CLIENTS (em3) — 10.10.30.1/24

## Issue: WAN/LAN interface swap
During initial setup, the interface assignment wizard mapped WAN and LAN
backwards (LAN was assigned to em0, WAN to em1), which pointed the
192.168.56.10 management address at the wrong physical NIC. This caused
the web GUI to be unreachable even though the address was configured
correctly. Fixed by re-running "Assign interfaces" and explicitly mapping
WAN -> em0, LAN -> em1, OPT1 -> em2, OPT2 -> em3.

## SERVERS interface rule
| Action | Proto | Source | Destination | Port | Description |
|---|---|---|---|---|---|
| Pass | any | SERVERS net | any | any | Servers may reach out |

## CLIENTS interface rules
| # | Action | Proto | Source | Destination | Port | Description |
|---|--------|-------|--------|-------------|------|-------------|
| 1 | Pass | UDP | CLIENTS net | 10.10.20.10 | 53 | DNS to DC01 |
| 2 | Pass | TCP | CLIENTS net | 10.10.20.10 | 53 | DNS to DC01 (large replies) |
| 3 | Pass | TCP | CLIENTS net | 10.10.20.10 | 88 | Kerberos auth |
| 4 | Pass | TCP | CLIENTS net | 10.10.20.10 | 464 | Kerberos password change |
| 5 | Pass | TCP | CLIENTS net | 10.10.20.10 | 389 | LDAP |
| 6 | Pass | TCP | CLIENTS net | 10.10.20.10 | 636 | LDAPS |
| 7 | Pass | TCP | CLIENTS net | 10.10.20.10 | 445 | SMB (group policy, shares) |
| 8 | Pass | TCP | CLIENTS net | 10.10.20.10 | 123 | NTP time sync |
| 9 | Block | any | CLIENTS net | SERVERS net | any | Deny all other server access |
| 10 | Pass | any | CLIENTS net | any | any | Internet allowed |

## Note on port splitting
The original guide listed Kerberos (88, 464) and LDAP/LDAPS (389, 636)
as single rules with comma-separated ports. OPNsense's "Single port or
range" field does not accept non-contiguous port lists, so each pair was
split into two separate rules (10 rules total instead of 8). Rule order
was preserved: all "pass to DC01" rules first, then the SERVERS-net block
rule, then the final internet-allow rule.

## Next steps
- Verify LDAPS rule uses port 636 (not 389, caught a copy-paste duplicate)
- Snapshot FW01 as `firewall-rules-done`
- Move on to Part 6: DC01 domain controller build