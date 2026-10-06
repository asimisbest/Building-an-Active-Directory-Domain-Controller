# Lab 3.1: Domain Controller Build and DNS Troubleshooting

Network Defense & Countermeasures (CSCI 4647), East Tennessee State University.

I promoted a hardened Windows Server 2022 VM to the first domain controller of an isolated Active Directory forest, hit a problem where the domain's DNS zones never appeared, and got it working on my third build. This repo holds the full write-up and every screenshot behind it.

## Start here

| File | What it is |
|---|---|
| [`WRITEUP.md`](WRITEUP.md) | The full step-by-step write-up, in order, with each screenshot inline |
| [`WRITEUP.docx`](WRITEUP.docx) | The same write-up as a Word file with the screenshots embedded |
| [`images/`](images/) | All 23 screenshots, numbered in the order the steps happened |

## Result at a glance

| Item | Status |
|---|---|
| Server promoted to domain controller | Done on all three attempts (`whoami` shows a domain account) |
| Domain DNS zones present | Only on attempt 3. Attempts 1 and 2 had none. |
| Domain records resolve (SRV, A, PTR) | Done, attempt 3 |
| Account-management auditing, Success + Failure | Done |
| OUs, test user, audit events 4720 and 4728 | Done |
| Firewall, Defender, core AD services | Verified |
| Final snapshot `03-DC-Healthy` | Started, not confirmed finished |

## Things to know before reading

- **The final forest name is `ad.etsu.lab`.** The assignment specified `ad.etsulab.example`. After the DNS problem my instructor said a `.local` suffix was acceptable as long as the lab worked. I meant to type `ad.etsulab.local` and typed `ad.etsu.lab` by mistake. Everything works under it, and I left it.
- **I don't know exactly why attempt 3 worked.** It changed the domain name and the planned post-reboot wait at the same time. The DNS Server log (events 708 and 4013) points at DNS starting before AD DS was ready, but I did not prove it.
- **Some evidence is missing.** Section 6 of the write-up lists it: no zone list or `dcdiag` for the working build, no screenshot of the security group, and an unconfirmed final snapshot.

## Evidence index

| # | File | What it shows |
|---|---|---|
| 01 | `01-attempt1-single-adapter.png` | NAT removed, one adapter left |
| 02 | `02-attempt1-pre-promotion-a.png` | Hostname, IP, DNS, firewall, Defender before promotion |
| 03 | `03-attempt1-pre-promotion-b.png` | AD DS, DNS, DHCP roles not yet installed |
| 04 | `04-attempt1-whoami-domain-admin.png` | Signed in as the domain administrator after promotion |
| 05 | `05-attempt1-wizard-installation.png` | Promotion wizard with the two expected warnings |
| 06 | `06-attempt1-dns-manager-no-zones.png` | DNS Manager with no domain zone |
| 07 | `07-attempt1-get-dnsserverzone.png` | Only default zones listed |
| 08 | `08-attempt1-dcdiag-dns-failed.png` | `dcdiag /test:DNS` reports failure |
| 09 | `09-attempt1-dns-event-708.png` | DNS found no zones at startup |
| 10 | `10-attempt1-dns-event-4013.png` | DNS waiting on AD DS synchronization |
| 11 | `11-attempt1-services-after-restart.png` | Services running after a full restart, zone still missing |
| 12 | `12-restore-clean-state.png` | Snapshot restored to the clean pre-promotion state |
| 13 | `13-attempt2-install-addsforest.png` | Second promotion, run in PowerShell |
| 14 | `14-attempt2-zones-missing.png` | Zones still missing after attempt 2 |
| 15 | `15-attempt2-get-dnsserverzone.png` | `Get-DnsServerZone` again, default zones only |
| 16 | `16-attempt2-ad-healthy.png` | AD healthy, DNS partitions exist in the directory |
| 17 | `17-attempt3-clean-state.png` | Clean state before the third build |
| 18 | `18-attempt3-promoted-whoami.png` | Promotion confirmed on the working build |
| 19 | `19-attempt3-dns-srv-ptr.png` | SRV, A, and PTR records resolve |
| 20 | `20-attempt3-audit-policy.png` | User and group management auditing, Success and Failure |
| 21 | `21-attempt3-aduc-structure.png` | Lab OUs and the test user |
| 22 | `22-attempt3-audit-events.png` | Events 4720 and 4728 in the Security log |
| 23 | `23-attempt3-final-checks-snapshot.png` | Final health checks and the snapshot in progress |

## Environment

Oracle VirtualBox on a Windows host. Windows Server 2022 Standard Evaluation (Desktop Experience), 2 vCPU, 4 GB RAM, 60 GB disk. Computer name `DC-LAB-01`. One host-only adapter on `10.20.2.0/24` at `10.20.2.10`, no gateway, DNS pointing at itself.
