# Lab 3.1 Full Write-up: Promoting a Windows Server 2022 VM to a Domain Controller

Course: Network Defense & Countermeasures (CSCI 4647), East Tennessee State University
Author: Mohamed Khalil

This is the step-by-step record of everything I did on this lab, in the order I did it, with the screenshot that backs up each step. It covers three build attempts. Attempts 1 and 2 promoted successfully but left the domain without its DNS zones. Attempt 3 worked.

How to read it: each step lists what I did, what I expected to see, and what I actually saw. Screenshot numbers match the files in `images/`.

## Contents

1. [Starting point](#1-starting-point)
2. [Attempt 1: Server Manager wizard (Sep 18)](#2-attempt-1-server-manager-wizard-sep-18)
3. [Troubleshooting attempt 1](#3-troubleshooting-attempt-1)
4. [Restore and Attempt 2: PowerShell promotion (Sep 18 to 21)](#4-restore-and-attempt-2-powershell-promotion-sep-18-to-21)
5. [Attempt 3: the working build (Oct 5)](#5-attempt-3-the-working-build-oct-5)
6. [What is not covered by my evidence](#6-what-is-not-covered-by-my-evidence)
7. [Command reference](#7-command-reference)

---

## 1. Starting point

I started from the VM I built in Lab 2.1: Windows Server 2022 Standard Evaluation (Desktop Experience), 2 vCPU, 4 GB RAM, 60 GB disk, in VirtualBox. It had two network adapters, a NAT adapter used for Windows Update and a host-only adapter on `LAB-LAN` (`10.20.2.0/24`, DHCP disabled), with the host-only side set to the static address `10.20.2.10`. Windows Update never completed on this VM (it hung repeatedly), which I documented in my Lab 2.1 submission.

Lab 3.1 turns this server into the first domain controller of a new, isolated forest.

## 2. Attempt 1: Server Manager wizard (Sep 18)

Forest name used: `ad.etsulab.example` (the name the assignment specified).

### Step 1. Remove the NAT adapter

A domain controller should be single-homed. A second adapter can make it register the wrong address in DNS, and the NAT path was only ever there for updates.

What I did: shut the VM down, turned off Adapter 1 (NAT) in VirtualBox under Settings, Network, then booted it again.

Expected: one network adapter left.
Observed: `Get-NetAdapter` shows a single adapter, `Ethernet`, index 10, status Up.

```powershell
Get-NetAdapter | Sort-Object ifIndex | Format-Table ifIndex, Name, Status, MacAddress
```

![Single adapter remains](images/01-attempt1-single-adapter.png)

### Step 2. Finalize networking and rename the server

What I did:
- Renamed the interface to `LAB-LAN`.
- Confirmed the static address `10.20.2.10/24` with a blank default gateway.
- Pointed DNS at the server itself (`10.20.2.10`), because it will host its own AD-integrated DNS.
- Renamed the computer to `DC-LAB-01` and rebooted.

```powershell
Set-DnsClientServerAddress -InterfaceAlias "LAB-LAN" -ServerAddresses 10.20.2.10
Rename-Computer -NewName "DC-LAB-01" -Restart
```

### Step 3. Verify the pre-promotion state

I ran the checks below before touching AD DS, so I'd have a known-clean starting point.

```powershell
hostname
Get-NetIPConfiguration
Get-DnsClientServerAddress -AddressFamily IPv4
Get-NetFirewallProfile | Select-Object Name, Enabled
Get-MpComputerStatus | Select-Object AntivirusEnabled, RealTimeProtectionEnabled
Get-WindowsFeature AD-Domain-Services,DNS,DHCP
```

Expected: hostname `DC-LAB-01`, `LAB-LAN` at `10.20.2.10` with no gateway, DNS pointing at itself, firewall on for all three profiles, Defender and real-time protection on, and the three roles showing `Available` (not installed).
Observed: all of the above.

![Pre-promotion state, part 1](images/02-attempt1-pre-promotion-a.png)

![Pre-promotion state, part 2: roles not installed](images/03-attempt1-pre-promotion-b.png)

### Step 4. Take the rollback snapshot

I shut the VM down cleanly and took a snapshot named `02-Pre-ADDS`. It paid for itself later, because I restored it twice. The VirtualBox title bar in the next screenshot shows `(02-Pre-ADDS)`.

### Step 5. Install AD DS and promote the server

What I did, in Server Manager: Manage, Add Roles and Features, role-based installation, checked Active Directory Domain Services (with management tools), installed it, then clicked the notification flag and chose "Promote this server to a domain controller." In the wizard I chose Add a new forest, entered `ad.etsulab.example`, kept DNS Server and Global Catalog checked, set a lab-only DSRM password, kept the default paths, ran the prerequisite check, and clicked Install.

Expected: two non-blocking warnings (an NT 4.0 cryptography note, and a DNS delegation warning because there is no parent DNS zone), then an automatic restart.
Observed: exactly those two warnings, with the installer on "Waiting for DNS installation to finish."

![AD DS configuration wizard, Installation page](images/05-attempt1-wizard-installation.png)

### Step 6. Sign in and confirm the promotion

After the restart I signed in as the domain administrator and ran `whoami`.

Expected: a domain-qualified name.
Observed: `etsulab\administrator`, so the server was now a domain controller.

![whoami shows a domain account](images/04-attempt1-whoami-domain-admin.png)

## 3. Troubleshooting attempt 1

### Step 7. Check DNS and find the problem

I opened DNS Manager to look for the domain's zones.

Expected: forward lookup zones `ad.etsulab.example` and `_msdcs.ad.etsulab.example`.
Observed: nothing listed under Forward Lookup Zones.

![DNS Manager with no domain zones](images/06-attempt1-dns-manager-no-zones.png)

I then checked from PowerShell. The forest existed, but `Get-DnsServerZone` returned only the built-in reverse zones and `TrustAnchors`. The DNS service itself was running.

```powershell
Get-ADForest | Select-Object Name, ForestMode, RootDomain
Get-DnsServerZone
Get-Service DNS
```

![Forest exists, domain zone missing](images/07-attempt1-get-dnsserverzone.png)

### Step 8. Run DCDiag's DNS test

```powershell
dcdiag /test:DNS /v
```

The summary reports `ad.etsulab.example failed test DNS`. For `DC-LAB-01` the Basic and Registration tests show FAIL, and the Dynamic update test shows a warning.

![DCDiag DNS test summary](images/08-attempt1-dcdiag-dns-failed.png)

### Step 9. Read the DNS Server event log

```powershell
Get-WinEvent -LogName "DNS Server" -MaxEvents 20 | Select-Object TimeCreated, Id, LevelDisplayName, Message | Format-List
```

Event 708: the DNS server did not detect any zones during initialization, so it ran as a caching-only server.

![DNS Server event 708](images/09-attempt1-dns-event-708.png)

Event 4013: DNS was waiting for Active Directory Domain Services to signal that its initial synchronization had finished.

![DNS Server event 4013](images/10-attempt1-dns-event-4013.png)

Diagnosis: DNS started before AD DS was ready, so it never loaded the zone from the directory. My working theory was a startup ordering problem made worse by a small VM (2 vCPU, 4 GB). I did not prove it.

### Step 10. Try the quick fixes

What I tried, in order:
1. `ipconfig /registerdns`, then `Restart-Service DNS`. The zone did not appear.
2. A full VM restart. The VM hung on the restart screen for a long time. After it stayed stuck I powered it off in VirtualBox and started it again.

After the restart ADWS, DFSR, DNS, and Netlogon were all running. (The red error is my typo of `KDS` for the real service name `Kdc`.) The domain zone was still missing.

![Services running after restart](images/11-attempt1-services-after-restart.png)

## 4. Restore and Attempt 2: PowerShell promotion (Sep 18 to 21)

### Step 11. Restore the snapshot

I restored `02-Pre-ADDS` to get back to a clean, pre-promotion state.

Expected: local administrator login, AD DS not installed.
Observed: `whoami` returns the local `...-01\administrator` account and AD DS shows `Available`.

![Back to the clean pre-promotion state](images/12-restore-clean-state.png)

### Step 12. Promote again, this time in PowerShell

```powershell
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
Install-ADDSForest -DomainName "ad.etsulab.example" -InstallDNS -Force
```

The command above is my reconstruction. The screenshot shows the output, not the command line I typed.

Expected: the same two warnings, then an automatic restart.
Observed: validation passed, "Installing new forest," and the same two warnings. The first boot after this promotion sat on "Applying computer settings" for a long time before reaching the login screen.

![Install-ADDSForest in PowerShell](images/13-attempt2-install-addsforest.png)

One thing I only noticed afterward: this attempt ended up with the NetBIOS name `AD` instead of `ETSULAB`, as the later `Get-ADDomain` output shows. In attempt 1 and attempt 3 the sign-in prefix is `etsulab`.

### Step 13. Check again after the restart

Same result. `whoami` shows a domain account, but `Get-DnsServerZone` lists only the default reverse zones.

A line in my PowerShell window reads `Rename-Computer -NewName "SRV-LAB-01" -Restart`. Renaming a domain controller after promotion breaks its identity in AD, so that would have been a serious mistake. The later `Get-ADDomain` output (screenshot 16) still shows `DC-LAB-01` as the controller, so the name did not change.

![Zones still missing after the second promotion](images/14-attempt2-zones-missing.png)

![Get-DnsServerZone again](images/15-attempt2-get-dnsserverzone.png)

### Step 14. Confirm AD itself is healthy

```powershell
Get-Service ADWS,DNS,DFSR,Kdc,Netlogon
Get-ADDomain
```

Observed: services running, and `Get-ADDomain` returns full domain data. Its SubordinateReferences list includes `DomainDnsZones` and `ForestDnsZones`, so the DNS partitions exist in the directory. DNS was not serving the zone from them.

![AD healthy, DNS zones still missing](images/16-attempt2-ad-healthy.png)

I stopped here for the day and prepared a partial submission, because I could not get past DNS.

## 5. Attempt 3: the working build (Oct 5)

### Step 15. Restore and rebuild

I restored `02-Pre-ADDS` again and confirmed the clean state: hostname `DC-LAB-01`, AD DS, DHCP, and DNS all `Available`.

![Clean state before attempt 3](images/17-attempt3-clean-state.png)

My instructor said a `.local` suffix would be fine as long as the lab worked. I meant to rebuild as `ad.etsulab.local` but typed **`ad.etsu.lab`**. I noticed after it worked and left it. This means attempt 3 differs from attempts 1 and 2 in two ways at once: the domain name, and my plan to wait longer after the reboot before signing in (I did not time the wait). I can't tell which one mattered. The command below is my reconstruction, since I have no screenshot of the command line itself.

```powershell
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
Install-ADDSForest -DomainName "ad.etsu.lab" -DomainNetbiosName "ETSULAB" -InstallDNS -Force
```

### Step 16. Confirm the promotion

Expected: `etsulab\administrator`, hostname `DC-LAB-01`.
Observed: both. Server Manager in the background shows the new roles.

![Promoted, signed in as the domain administrator](images/18-attempt3-promoted-whoami.png)

### Step 17. Validate DNS

I resolved the domain's records from the DC itself.

```powershell
Resolve-DnsName -Type SRV _ldap._tcp.dc._msdcs.ad.etsu.lab
Resolve-DnsName 10.20.2.10
```

Expected: the LDAP SRV record pointing at this DC, and the reverse lookup returning its name.
Observed: the SRV record returns `dc-lab-01.ad.etsu.lab` on port 389 with the A record `10.20.2.10`, and the reverse lookup for `10.20.2.10` returns `DC-LAB-01.ad.etsu.lab`. The red error at the top is my typo (a capital `I` in `_ldap`). The correctly typed command succeeded. Since the reverse lookup resolves, I infer a reverse zone for `10.20.2` exists, but I did not capture a screenshot of it.

![SRV, A, and PTR resolution](images/19-attempt3-dns-srv-ptr.png)

### Step 18. Turn on account-management auditing

I did this before creating any accounts so the events would be recorded. In Group Policy Management I created a GPO named `DC-Audit-Baseline` linked to the Domain Controllers OU, and under Advanced Audit Policy Configuration, Account Management, I enabled **Audit User Account Management** and **Audit Security Group Management** for Success and Failure.

Expected: both subcategories show "Success and Failure."
Observed: both do.

![Audit policy settings](images/20-attempt3-audit-policy.png)

### Step 19. Build the OU, user, and group structure

In Active Directory Users and Computers I created three OUs at the domain root (`Lab Users`, `Lab Groups`, `Lab Computers`), a global security group, and a fictional test user `ND Student01` in `Lab Users`, then added the user to the group.

Observed: the three OUs and the user are visible under `ad.etsu.lab`. The group sits in `Lab Groups` and isn't shown in this screenshot, so the next step's events are what confirm the membership change.

![Active Directory Users and Computers](images/21-attempt3-aduc-structure.png)

### Step 20. Confirm the audit events fired

```powershell
Get-WinEvent -FilterHashtable @{ LogName='Security'; Id=4720,4728 } -MaxEvents 10 |
  Select-Object TimeCreated, Id, Message
```

My first attempts failed on a syntax error (I left out the `;` between hashtable entries). The corrected command returned results.

Expected: 4720 (user account created) and 4728 (member added to a security-enabled global group).
Observed: both event IDs, several entries dated 10/5/2026. Two older entries dated 9/17 also appear and I haven't explained them.

![Security events 4720 and 4728](images/22-attempt3-audit-events.png)

### Step 21. Final health check and snapshot

```powershell
Get-NetFirewallProfile | Select-Object Name, Enabled
Get-MpComputerStatus | Select-Object AntivirusEnabled, RealTimeProtectionEnabled
Get-ADDefaultDomainPasswordPolicy | Select-Object MinPasswordLength, ComplexityEnabled, LockoutThreshold
Get-Service ADWS,DNS,DFSR,KDC,Netlogon
```

Observed: firewall enabled on Domain, Private, and Public; Defender and real-time protection on; minimum password length 7, complexity on, lockout threshold 0 (the defaults, left unchanged); ADWS, DNS, DFSR, KDC, and Netlogon all running. (I mistyped `Get-MpComputerStatus` once. The corrected command is below it.)

I then started a snapshot for the finished state, `03-DC-Healthy`. The screenshot catches VirtualBox at 47% and the title bar reads "Taking Online Snapshot," meaning the VM was still running instead of cleanly shut down first.

![Final checks and snapshot in progress](images/23-attempt3-final-checks-snapshot.png)

## 6. What is not covered by my evidence

I'd rather list these than leave them to be discovered:

- **The forest name doesn't match the assignment.** The spec says `ad.etsulab.example`. The working build is `ad.etsu.lab`, a typo of my intended `ad.etsulab.local`.
- **No DNS Manager screenshot of the working build.** I confirmed DNS through `Resolve-DnsName`. I did not capture the zone list, the reverse zone, or the `_msdcs` zone.
- **No `dcdiag` output for the working build.** The only DCDiag screenshot is the failing one from attempt 1.
- **The security group isn't visible.** The ADUC screenshot shows the OUs and user. The 4728 events indicate group membership changes happened.
- **The final snapshot is unconfirmed.** I never captured the Snapshots pane showing `01-Baseline-Ready`, `02-Pre-ADDS`, and `03-DC-Healthy` together, and the last snapshot was an online snapshot that I didn't see finish.
- **The root cause of the DNS failure is unknown.** Events 708 and 4013 suggest DNS started before AD DS was ready. The fix on attempt 3 coincided with a changed name and a longer wait, so I can't credit either.
- **I never tried `dnscmd /zoneadd`.** Forcing the zone into existence is something I'd try before restoring a snapshot next time.

## 7. Command reference

```powershell
# Networking and identity
Get-NetAdapter | Sort-Object ifIndex | Format-Table ifIndex, Name, Status, MacAddress
Set-DnsClientServerAddress -InterfaceAlias "LAB-LAN" -ServerAddresses 10.20.2.10
Rename-Computer -NewName "DC-LAB-01" -Restart

# Promotion
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
Install-ADDSForest -DomainName "<domain>" -DomainNetbiosName "ETSULAB" -InstallDNS -Force

# Health and DNS checks
whoami
Get-ADForest | Select-Object Name, ForestMode, RootDomain
Get-ADDomain
Get-DnsServerZone
Get-Service ADWS,DNS,DFSR,Kdc,Netlogon
dcdiag /test:DNS /v
Get-WinEvent -LogName "DNS Server" -MaxEvents 20 | Select-Object TimeCreated, Id, LevelDisplayName, Message | Format-List
Resolve-DnsName -Type SRV _ldap._tcp.dc._msdcs.<domain>
Resolve-DnsName 10.20.2.10

# Audit evidence
auditpol /get /subcategory:"User Account Management"
auditpol /get /subcategory:"Security Group Management"
Get-WinEvent -FilterHashtable @{ LogName='Security'; Id=4720,4728 } -MaxEvents 10 | Select-Object TimeCreated, Id, Message
```
