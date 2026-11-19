# SAM-ADM-003: Automated lab diagrams

| Field | Value |
| --- | --- |
| Document ID | SAM-ADM-003 |
| Title | Generating lab diagrams automatically from Proxmox, Active Directory and the network |
| Owner / Author | Techworks |
| Status | In progress. Part A to Part D built and verified on the first Proxmox host on 2026-11-17. Later parts are planned (see Open items) |
| Last updated | 2026-11-17 |
| Depends on | A Linux server that can reach each Proxmox host on port 8006 (the Ubuntu DevOps server) |

**Search keywords:** `pvesh`, `pvecm status`, `corosync.conf does not exist`, `/cluster/resources`, `API token`, `PVEAuditor`, `Privilege Separation`, `pveum`, `Mermaid`, `diagram as code`, `read-only API`

## How to read this document

| If you are | Read |
| --- | --- |
| Level 1 / Level 2 technician | Purpose, Prerequisites, the **Steps** and **Expected result** in each Part, Troubleshooting, Command reference |
| Systems administrator / engineer | All of the above, plus the **Why** notes and **Automation equivalent** |
| Architect / reviewer | Purpose, Design notes, Open items |

Conventions:

- Click paths are written one click per line, each starting with `>`.
- Every step states an **Expected result**. If you do not see it, stop and go to Troubleshooting.
- **Roadblock** notes record problems actually hit, with cause and fix.
- Commands under **Automation equivalent** do the same as the manual steps. Unless marked otherwise, they have not been run in this lab.

---

## 1. Purpose and outcome

Produce diagrams of the lab from the live environment rather than drawing them by hand, so they stay accurate. A script reads each system, writes a Mermaid diagram, and GitHub displays it inside the repository's Markdown files.

This document covers the first source, Proxmox: what its API returns, and how to give a script safe, read-only access to it.

**You know it works when** (for the parts complete so far):

- `pvesh get /cluster/resources --type vm` on a Proxmox host lists that host's VMs.
- The host has a user `diagram@pve` with an API token `diagram-script`, and that token can read but not change anything.

## 2. Scope and environment

| Item | Value |
| --- | --- |
| Proxmox hosts | Three standalone hosts (`pve`, `pve02`, `pve03`). **Not** a cluster |
| Script host | The Ubuntu DevOps server |
| Access method | Proxmox REST API on port 8006, with a per-host API token |
| Output format | Mermaid, rendered by GitHub |

Out of scope for this part: the Active Directory and network diagrams (planned, see Open items).

## 3. Background: what the diagram is built from

The Proxmox API returns one block per VM or container. These fields become the diagram:

| Field | Example | Becomes |
| --- | --- | --- |
| `node` | `pve` | Which host box the VM sits inside |
| `name`, `vmid` | `pfsense`, `100` | The VM's label |
| `type` | `qemu` (VM) or `lxc` (container) | Shape or label |
| `status` | `running` / `stopped` | Colour |
| `maxmem` | `2147483648` | Memory, in **bytes** (divide by 1,073,741,824 for GB) |
| `maxcpu` | `2` | vCPU count |
| `tags` | `ubuntu01;qapp` | Optional grouping. Set in Proxmox, separated by `;` |

**Standalone hosts vs a cluster:** in a cluster, one query returns every VM on every host. Standalone hosts each know only their own VMs and keep their own users, so the script queries each host separately, and the access set-up in Parts B and C is repeated on each host.

## 4. Prerequisites

- Administrator access to each Proxmox web interface (`https://<pve-address>:8006`).
- A password manager or other private place to store an API token secret.

---

## Part A: Explore the API by hand

**Outcome:** you know whether the hosts are clustered and what data the API returns.

**Steps:**

```
Browser
> https://<pve-address>:8006
> sign in
> left pane: Datacenter
> click the node (pve)
> Shell
```

```
pvecm status
pvesh get /cluster/resources --type vm --output-format json-pretty | head -40
```

**Expected result:**

- `pvecm status` on a standalone host prints: `Error: Corosync config '/etc/pve/corosync.conf' does not exist - is this node part of a cluster?` This is not a fault; it confirms the host is standalone.
- `pvesh get` prints a list of blocks, one per VM, containing the fields in section 3.

**Why:** `pvesh` is a command-line client for the same API the web interface uses. Anything readable with `pvesh` on the host can be read by a script over the network, so this shows exactly what the script will receive.

## Part B: Create a dedicated read-only user

**Outcome:** a user that exists only for the diagram script.

**Steps** (on each Proxmox host):

```
Datacenter
> Permissions
> Users
> Add
> User name: diagram
> Realm: Proxmox VE authentication server
> Comment: Read-only account for the diagram script
> leave Enabled ticked
> Add
```

If a password is requested, set a long random one and do not keep it. The account is used only through its API token.

**Expected result:** the Users list shows `diagram` with Realm `pve`, beside `root` with Realm `pam`.

**Why:** a script should never use the root password. A dedicated account can be audited, limited and removed without affecting anything else. The `pve` realm stores the user in Proxmox's own database; it is not a Linux account on the host.

**Automation equivalent:**

```
pveum user add diagram@pve --comment "Read-only account for the diagram script"
```

> **Roadblock:** the user was first created as `digram`. Proxmox cannot rename a user. **Fix:** select the row, Remove, and add it again with the correct name. Do this before creating a token, because the user name is part of the token ID used by the script. Check the selected row before clicking Remove: removing `root@pam` would lock you out of the web interface.

## Part C: Create an API token

**Outcome:** a token the script presents instead of a password.

**Steps:**

```
Datacenter
> Permissions
> API Tokens
> Add
> User: diagram@pve
> Token ID: diagram-script
> leave Privilege Separation ticked
> Expire: never
> Comment: Diagram script on the Ubuntu DevOps server
> Add
```

A window shows two values:

| Field | Example | Secret? |
| --- | --- | --- |
| Token ID | `diagram@pve!diagram-script` | No |
| Secret | `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` | **Yes. Shown only once** |

**Copy the Secret before closing the window** and store it privately. It cannot be displayed again.

**Expected result:** the API Tokens list shows user `diagram@pve`, Token ID `diagram-script`, Privilege Separation `Yes`.

**Why:**

- **Privilege Separation** means the token receives only the permissions assigned to the token itself, not those of its user. It is the stricter setting.
- A token can be revoked on its own without touching the user, and its use is identifiable in logs.
- Never paste the secret into a chat, a ticket or the repository. The repository's pre-commit hook does not recognise the Proxmox token format, so it would not catch one.

**Automation equivalent:**

```
pveum user token add diagram@pve diagram-script --privsep 1 --comment "Diagram script on the Ubuntu DevOps server"
```

This prints the secret once, in the same way.

## Part D: Give the token read-only access

**Outcome:** the token can read everything on the host and change nothing.

**Steps:**

```
Datacenter
> Permissions
> Add
> API Token Permission
> Path: /
> API Token: diagram@pve!diagram-script
> Role: PVEAuditor
> leave Propagate ticked
> Add
```

**Expected result:** the Permissions list shows a row with Path `/`, User/Group/API Token `diagram@pve!diagram-script`, Role `PVEAuditor`. The Propagate column (`true`) may be off-screen to the right on a narrow window.

**Why:**

- With Privilege Separation, a new token has **no** permissions until one is granted to the token itself. Granting the role to the user would not help.
- **`PVEAuditor`** is Proxmox's built-in read-only role: it can view hosts, VMs, storage and settings, but cannot start, stop, change or delete anything.
- **Path `/`** is the top of the host's permission tree. **Propagate** applies the permission to everything below it, so VMs created later are covered automatically.

**Automation equivalent:**

```
pveum acl modify / --tokens 'diagram@pve!diagram-script' --roles PVEAuditor
```

Use single quotes: in Bash, `!` inside double quotes triggers history expansion and the command fails or does something unexpected.

---

## 6. Verification

On the Proxmox host shell:

```
pveum user list
pveum user token list diagram@pve
pveum acl list
pveum user token permissions diagram@pve diagram-script
```

**Expected result:**

- `diagram@pve` is listed as enabled, and its token list contains `diagram-script` with `privsep` `1`.
- `pveum acl list` has a row for `/`, type `token`, `diagram@pve!diagram-script`, role `PVEAuditor`, propagate `1`.
- The token permissions list shows only `.Audit` privileges (for example `VM.Audit`, `Sys.Audit`, `Datastore.Audit`), and nothing ending in `.Allocate`, `.Modify` or `.PowerMgmt`.

## 7. Rollback

| To undo | Do this |
| --- | --- |
| The permission | Datacenter > Permissions > select the `/` row for `diagram@pve!diagram-script` > Remove. Or `pveum acl delete / --tokens 'diagram@pve!diagram-script' --roles PVEAuditor` |
| The token | Datacenter > Permissions > API Tokens > select `diagram@pve!diagram-script` > Remove. Or `pveum user token remove diagram@pve diagram-script` |
| The user | Datacenter > Permissions > Users > select `diagram` > Remove. Or `pveum user delete diagram@pve`. Removing the user also removes its tokens |

## 8. Troubleshooting

| Symptom | Likely cause | Check | Fix |
| --- | --- | --- | --- |
| `pvecm status`: "Corosync config ... does not exist" | The host is standalone, not clustered | Expected on standalone hosts | None. Query each host separately |
| `pvesh get` lists only some VMs | The host is standalone; it only knows its own VMs | `pvecm status` | Run the query on each host |
| Token secret lost | It is shown only once | n/a | Remove the token and create a new one |
| User name misspelled | Typing error | Users list | Remove and re-create before creating the token |
| API returns `403 Forbidden` / "Permission check failed" | Token has no permission, or it was given to the user instead of the token | `pveum acl list` | Add an **API Token Permission** (Part D) for `diagram@pve!diagram-script` |
| `bash: !diagram: event not found` | `!` in double quotes triggers Bash history expansion | n/a | Put the token ID in single quotes |
| `diagram` user not in the API Tokens "User" list | The user was created on a different host, or not saved | Users list on this host | Create the user on this host first |

## 9. Key lessons

> ### A script gets its own account and a read-only token, never the root password.

- **Standalone hosts each need their own set-up.** Users and tokens are per host unless the hosts are clustered.
- **A secret shown once must be captured immediately**, and stored outside the repository.
- **Fix naming mistakes before anything depends on the name.**
- **With Privilege Separation, grant permissions to the token, not the user.** Least privilege: the role is `PVEAuditor`, which can look but not touch.

## 10. Command reference

| Task | Command |
| --- | --- |
| Is this host in a cluster? | `pvecm status` |
| List VMs and containers on this host | `pvesh get /cluster/resources --type vm --output-format json-pretty` |
| Show only the first 40 lines of output | `<command> \| head -40` |
| Create a Proxmox user | `pveum user add diagram@pve --comment "<text>"` |
| List Proxmox users | `pveum user list` |
| Delete a Proxmox user | `pveum user delete <user>@pve` |
| Create an API token with privilege separation | `pveum user token add diagram@pve diagram-script --privsep 1` |
| List a user's tokens | `pveum user token list diagram@pve` |
| Remove a token | `pveum user token remove diagram@pve diagram-script` |
| Grant the token read-only access to everything | `pveum acl modify / --tokens 'diagram@pve!diagram-script' --roles PVEAuditor` |
| List all permissions on the host | `pveum acl list` |
| Show what a token is actually allowed to do | `pveum user token permissions diagram@pve diagram-script` |
| Remove the token's permission | `pveum acl delete / --tokens 'diagram@pve!diagram-script' --roles PVEAuditor` |

## 11. Design notes

- **Per-host tokens.** Three standalone hosts mean three users and three tokens. Clustering the hosts would reduce this to one, but clustering has its own requirements (stable networking, quorum) and is out of scope here.
- **Token storage.** The secret will be kept on the script host in a file outside the repository, readable only by the account that runs the script.
- **TLS.** The Proxmox hosts use self-signed certificates by default. The script will need either the hosts' certificate authority installed or certificate checking relaxed; trusting the certificate is the better practice.

## 12. Open items

- [x] Part D: grant the token the `PVEAuditor` role on path `/` (done on `pve`, 2026-11-17).
- [ ] Part E: store the token secret on the Ubuntu DevOps server and test a read from the API with `curl`.
- [ ] Repeat Parts B to E on `pve02` and `pve03`.
- [ ] Part F: write the Proxmox diagram script (Mermaid output, IP addresses replaced with placeholders).
- [ ] Active Directory diagram: OUs and GPO links.
- [ ] Network diagram: firewall, switches, VLANs.
- [ ] Add the generated diagrams to the README.

## Document history

| Date | Change | By |
| --- | --- | --- |
| 2026-11-17 | First issue: Parts A to C | Techworks |
| 2026-11-17 | Added Part D (read-only permission), verification, rollback and commands | Techworks |
