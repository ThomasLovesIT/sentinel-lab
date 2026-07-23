## DC01 — Domain Controller (Part 6)

- Created DC01 VM: Windows Server 2025 Standard Evaluation (Server Core), 2048MB RAM, 40GB disk
- Network: servers-net (Internal Network), static IP 10.10.20.10
- Static IP configured via sconfig (gateway 10.10.20.1, DNS 10.10.20.10 self-referencing)
- Verified connectivity: ping 10.10.20.1 (FW01) successful
- Installed AD-Domain-Services role
- Promoted DC01 to Active Directory forest: sentinel.lab (NetBIOS: SENTINEL)
- Verified via Get-ADDomain

### Issues encountered
- First Install-ADDSForest attempt failed: "Name change pending. A reboot is required." Resolved with Restart-Computer, then reran the same command successfully.
- Hit intermittent VirtualBox keyboard input issues during login (dropped/duplicate keystrokes caused false "incorrect password" errors on a previous DC01 build). Rebuilt DC01 from scratch to rule out state corruption; second build completed without repeat issues.

## Session Update — 2026-07-24

- DC01 fully configured: AD forest (sentinel.lab), DHCP scope + options, DNS forwarding, WinRM enabled
- Created LabUsers OU with 5 test accounts (labuser1-5) for later DR drill (Part 13)
- DC01 shut down cleanly and snapshotted in VirtualBox (dc-promoted)
- Part 6 complete end-to-end
