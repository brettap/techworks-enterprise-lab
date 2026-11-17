# Techworks Enterprise Lab

A home lab that runs like a small company. Three simulated employees sign in every weekday morning, type, work with files on department shares, open an accounting system, and occasionally try to get into folders they shouldn't. Everything they do is audited, and the problems that come up are investigated and documented the way a support team would.

This repository holds the build documentation, scripts, and sample data for that environment.

> **Status:** active. Windows user simulation, file shares with auditing, and desktop accounting are built and verified. A fault-injection engine and Red Hat Enterprise Linux integration are next. See [Roadmap](#roadmap).

---

## What this demonstrates

| Area | What was done | Where |
| --- | --- | --- |
| Active Directory | OU design, users, AGDLP security groups, a dedicated share-admin group | [SAM-ADM-001](docs/SAM-ADM-001-Simulating-winuser-activity.md) |
| Group Policy | Computer vs user scope, item-level targeting, Group Policy Preferences, troubleshooting with `gpresult` and event 4098 | SAM-ADM-001 |
| Security auditing | Advanced Audit Policy, folder SACLs, reading events 4663, 4656 and 5145, saved Event Viewer views | SAM-ADM-001, [SAM-ADM-002](docs/SAM-ADM-002-GnuCash-shared-book-and-file-locks.md) |
| Automation | PowerShell remoting, scheduled tasks, VM power scheduling with `vmrun`, DPAPI-protected secrets | SAM-ADM-001 |
| File services | NTFS vs share permissions, access-based enumeration, drive maps by group, UAC behaviour on servers | SAM-ADM-001 |
| Application support | Deploying desktop accounting software, importing data, diagnosing and clearing stale file locks from evidence | SAM-ADM-002 |
| Linux administration | apt signing-key rotation, fingerprint verification | [SAM-016](docs/SAM-016-NO_PUBKEY-error.md) |
| Documentation | A layered runbook standard written for Level 1 technicians through architects | [Documentation standard](#documentation-standard) |

## Lab architecture

| Component | Platform | Role |
| --- | --- | --- |
| `Techworks-DC01`, `DC-02` | Windows Server 2025 on Proxmox | Domain controllers, DNS, file shares, audit log |
| `TW-CL00`, `TW-CL01`, `TW-CL02` | Windows 11 on VMware Workstation | Simulated employee workstations |
| Linux servers | Ubuntu on Proxmox | DevOps tooling |
| Network | Netgate (pfSense), UniFi switching and wireless | Firewall, VLANs, segmentation |

<!-- Add diagrams/lab-architecture.png here -->

### A simulated weekday

| Time | What happens |
| --- | --- |
| 8:00 AM | Scheduled task on the VMware host starts the three clients, staggered |
| 8:02 AM | Each client signs in as its employee (Finance, GIS, Operations) and maps its department drive |
| 8:03 AM onward | Each employee types in Notepad, then creates, reads or edits a file on their share. Now and then they try another department's share and are refused |
| All morning | The file server records every access and every refusal |
| 12:00 PM | The clients shut down cleanly |

## Documents

| ID | Title | Type |
| --- | --- | --- |
| [SAM-ADM-001](docs/SAM-ADM-001-Simulating-winuser-activity.md) | Simulating Windows user activity | Build guide |
| [SAM-ADM-002](docs/SAM-ADM-002-GnuCash-shared-book-and-file-locks.md) | GnuCash shared book and file locks | Build guide and support scenario |
| [SAM-016](docs/SAM-016-NO_PUBKEY-error.md) | apt NO_PUBKEY after a vendor key rotation | Incident fix |

## Documentation standard

Every document here follows the same structure, so that each reader can stop at the depth they need without the content being simplified:

| Section | Written for |
| --- | --- |
| Purpose and outcome, with a "you know it works when" test | Everyone |
| Numbered steps, one click per line, each with an expected result | Level 1 and Level 2 |
| A "why" for each step, set apart so it can be skipped | Engineers |
| Roadblocks: problems actually hit, with cause, proof and fix | Everyone |
| Verification, rollback and a symptom-based troubleshooting table | Level 1 and Level 2 |
| Automation equivalent, shown after the manual method | Engineers |
| Design notes and open items | Architects and reviewers |

**Naming:** `SAM-<nnn>` for administration plans and incident write-ups, `SAM-ADM-<nnn>` for the Windows administration lab series, `RB-<Platform>-<nnn>` for runbooks.

## Repository layout

```
.
├── README.md
├── docs/                 SAM and SAM-ADM documents
├── runbooks/             RB-<Platform>-<nnn> runbooks
├── scripts/
│   ├── powershell/       Activity script, VM power scripts, deployment snippets
│   └── bash/             Linux maintenance scripts
├── data/                 Sample data (fictional company and customers)
├── diagrams/             Architecture and flow diagrams
└── CHANGELOG.md
```

## Roadmap

- [x] Windows user activity simulation
- [x] Department shares, permissions and auditing
- [x] Desktop accounting application and file-lock support scenario
- [ ] Fault-injection engine: a database of support tickets from Level 1 to Level 3, injected on a schedule, each with a verify and rollback script
- [ ] Red Hat Enterprise Linux joined to Active Directory
- [ ] Web-based ERP (Odoo Community) with directory logins
- [ ] Central log collection
- [ ] Ansible for configuration across Windows and Linux

## How this was built

Built and operated by hand in my own lab. I used Claude (Anthropic) as a mentor throughout: to plan each phase, explain concepts, and help write the scripts and documentation. Every step was carried out and verified in the lab, and the roadblocks in each document are problems I actually ran into and worked through.

## Notes

- This is a lab. Auto logon, file services on a domain controller, and similar shortcuts are documented as such in each document's design notes.
- All company, customer and employee names are fictional.
- No passwords, keys or secrets are stored in this repository.
