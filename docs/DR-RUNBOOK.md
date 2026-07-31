# Runbook: Recovering a deleted Organizational Unit

**Severity:** Critical — all user authentication down, Windows and Linux
**Estimated recovery time:** 35–60 minutes
**Prerequisites:** DSRM password, system state backup newer than the deletion

---

## Detection

You will not receive an alert for this. Monitoring watches infrastructure health, and the
infrastructure is healthy — the domain controller is up, responding, and serving an empty
directory.

What you will see instead:

- A cluster of "cannot log in" tickets arriving within minutes of each other
- `Get-ADUser -Filter * -SearchBase "OU=<name>,DC=sentinel,DC=lab"` returns
  *directory object not found*
- On domain-joined Linux hosts, `id <username>` returns *no such user* once the SSSD cache expires

The skill here is pattern recognition: several simultaneous login failures are a directory
problem, not several coincidental account problems. Check the directory before troubleshooting
individual accounts.

---

## Decide which recovery path applies

| Condition | Path | Time |
|---|---|---|
| AD Recycle Bin was enabled **before** the deletion | `Restore-ADObject` | ~2 minutes |
| AD Recycle Bin was not enabled | DSRM authoritative restore | ~35–60 minutes |

The deciding factor is preparation, not the number of objects lost. The Recycle Bin is not
retroactive — enabling it after a deletion does not help with that deletion.

### Fast path (Recycle Bin enabled)

```powershell
Get-ADObject -Filter 'Name -like "labuser*"' -IncludeDeletedObjects | Restore-ADObject
```

Deleted objects remain recoverable for 180 days from the moment of deletion.

---

## Slow path: DSRM authoritative restore

Directory Services Restore Mode boots Windows with Active Directory offline. This is required
because the AD database cannot be overwritten while it is open and in use.

**A system state restore replaces the entire directory database.** Every change made between the
backup and the failure is lost, not only the deletion. Account this against your RPO before
proceeding.

### 0. Take a VM snapshot first

Non-negotiable in a lab, and the equivalent in production is a fresh backup of the current state.
An interrupted system state recovery leaves the OS and the AD database inconsistent and the
machine will not boot.

### 1. Enter DSRM

```powershell
bcdedit /set safeboot dsrepair
shutdown -r -t 0
```

Log in as `.\Administrator` — the leading `.\` selects the local account. The domain account will
not work, because the domain is not running. The sign-in screen should read **Sign in to: DC01**.

If the prompt asks for `SENTINEL\Administrator`, press ESC, choose *Other user*, and re-enter with
the `.\` prefix.

### 2. Identify the backup

```powershell
wbadmin get versions
```

Record the version identifier exactly as printed. **It is in UTC and will not match the local
backup time.** Do not adjust it — the restore will not find the backup.

### 3. Restore the system state

```powershell
wbadmin start systemstaterecovery -version:<VERSION> -authsysvol -quiet
```

Runs 10–20 minutes. It will appear frozen for long stretches. **Do not touch the console.**
Interrupting this is the most likely way to end up with an unbootable domain controller.

Detach any installer ISO from the VM beforehand, so that a "press any key to boot from CD" prompt
cannot interfere with the automatic reboot.

The machine reboots into DSRM again, because the safeboot flag is still set. This is expected.

### 4. Mark the restored objects authoritative

Restoring the database alone is not sufficient. Active Directory versions every change, and a
deletion is a change. Without this step the restored objects can be treated as stale and the
deletion re-applied — which is exactly what happens in a multi-DC environment when the deletion
replicates back from a peer.

```
ntdsutil
activate instance ntds
authoritative restore
restore subtree OU=LabUsers,DC=sentinel,DC=lab
```

Confirm when prompted. Record the number of objects marked. Then `quit`, `quit`.

### 5. Return to normal boot

```powershell
bcdedit /deletevalue safeboot
shutdown -r -t 0
```

### 6. Verify

On DC01:

```powershell
Get-ADUser -Filter * -SearchBase "OU=LabUsers,DC=sentinel,DC=lab" | Select Name
```

On each domain-joined Linux host:

```bash
sudo systemctl restart sssd
id labuser1
su - labuser1
```

The SSSD restart clears the negative cache. Without it, clients will keep reporting the users as
missing for some minutes after the directory is healthy again.

---

## Post-incident actions

- Re-enable `ProtectedFromAccidentalDeletion` on the restored OU
- Enable the AD Recycle Bin so this scenario never requires DSRM again:

  ```powershell
  Enable-ADOptionalFeature 'Recycle Bin Feature' -Scope ForestOrConfigurationSet `
    -Target 'sentinel.lab' -Confirm:$false
  ```

  This is a one-time, irreversible, forest-wide change. It covers every object automatically and
  requires no ongoing maintenance. It protects only objects deleted after it is enabled.
- Ship Directory Service event logs to Loki and alert on event ID 5141 (directory object deleted),
  closing the detection gap described above
- Review which accounts hold rights to delete OUs

---

## Lessons learned

- The AD Recycle Bin turns a 35-minute outage into a 2-minute fix, and costs nothing to enable.
  The only reason to know the DSRM procedure is that you will eventually inherit an environment
  where nobody turned it on.
- Monitoring did not detect this failure and could not have. Infrastructure metrics do not
  observe logical data loss.
- An interrupted restore is worse than no restore. Snapshot before destructive work.
