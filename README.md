# Project SENTINEL

A segmented small-business network built with VirtualBox, OPNsense, Windows Server 2025, and Ubuntu Server — designed as a hands-on portfolio project for network/sysadmin roles.

## Status: In Progress

Completed the domain controller, monitoring/automation host, and Linux client. Network segmentation is verified and working.

## Progress

- **FW01 (OPNsense):** firewall rules and interfaces configured; segmentation rules enforced.
- **DC01 (Windows Server 2025):** Active Directory forest `sentinel.lab`, DNS, DHCP, WinRM, and five test users.
- **MON01 (Ubuntu):** Docker and Ansible installed; monitoring stack to follow.
- **CLIENT01 (Ubuntu):** built on `clients-net`; segmentation confirmed.

## Segmentation verified

CLIENT01 can reach DC01 for domain services (DNS, Kerberos) but is blocked from MON01. Confirmed by ping (0% loss to DC01, 100% loss to MON01) and a timed-out SSH attempt to MON01, blocked by the CLIENTS deny rule.

## Next

DHCP relay (Part 9): CLIENT01 will pull its address automatically via a relay through FW01, since the DHCP server (DC01) sits on a separate segment.