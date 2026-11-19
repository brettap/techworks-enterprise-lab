# SAM-ADM-003: Automated lab diagrams

| Field | Value |
| --- | --- |
| Document ID | SAM-ADM-003 |
| Title | Generating lab diagrams automatically from Proxmox, Active Directory and the network |
| Owner / Author | Techworks |
| Status | Proxmox source complete (Parts A to G, all three hosts, 2026-11-18). Active Directory and network diagrams deferred (see Open items) |
| Last updated | 2026-11-19 |
| Depends on | A Linux server that can reach each Proxmox host on TCP 8006 (the Ubuntu DevOps server) |

**Search keywords:** `pvesh`, `pvecm status`, `corosync.conf does not exist`, `/cluster/resources`, `API token`, `PVEAuditor`, `Privilege Separation`, `pveum`, `data: []`, `PVEAPIToken`, `curl -H @-`, `read -rs`, `CERTIFICATE_VERIFY_FAILED`, `IP address mismatch`, `pvecm updatecerts --force`, `pve-root-ca.pem`, `search domain`, `Connection closed (Code: 1006)`, `Mermaid`, `diagram as code`

## How to read this document

| If you are | Read |
| --- | --- |
| Level 1 / Level 2 technician | Purpose, Prerequisites, the **Steps** and **Expected result** in each Part, Troubleshooting, Command reference |
| Systems administrator / engineer | All of the above, plus the **Why** notes and **Automation equivalent** |
| Architect / reviewer | Purpose, Background, Design notes, Open items |

Conventions:

- Click paths are written one click per line, each starting with `>`.
- Every step states an **Expected result**. If you do not see it, stop and go to Troubleshooting.
- **Roadblock** notes record problems actually hit, with cause and fix.
- Commands under **Automation equivalent** do the same as the manual steps. Unless marked otherwise, they have not been run in this lab.
- Placeholders: `<pve-address>`, `<pve02-address>`, `<pve03-address>` for host addresses; `<script-user>@<devops-server>` for the account and server that run the script. Real values never go in this document or the repository.

---

## 1. Purpose and outcome

Produce diagrams of the lab from the live environment rather than drawing them by hand, so they stay accurate. A script reads each system, writes a Mermaid diagram, and GitHub displays it inside the repository's Markdown files.

This document covers the first source, Proxmox: what its API returns, how to give a script safe, read-only access to it over verified TLS, and the script itself.

**You know it works when:**

- Each Proxmox host has a user `diagram@pve` with an API token `diagram-script` that can read but not change anything.
- From the Ubuntu DevOps server, `python3 tools/diagrams/pve_to_mermaid.py > /dev/null` prints a guest count for every host, with no `failed` line and no `--insecure` flag.
- `docs/diagrams/proxmox-guests.mmd` shows one box per host and one node per guest.

The generated diagram is a **current-state inventory**: what exists and where. The target architecture (how networks connect, what talks to what) is drawn by hand and kept alongside it (see Design notes).

## 2. Scope and environment

| Item | Value |
| --- | --- |
| Proxmox hosts | Three standalone hosts (`pve`, `pve02`, `pve03`). **Not** a cluster |
| Script host | The Ubuntu DevOps server |
| Access method | Proxmox REST API on TCP 8006, with a per-host API token |
| TLS | Verified against each host's own CA |
| Output format | Mermaid, rendered by GitHub |
| Lab domain | `techworks.local` |

Out of scope for this issue: the Active Directory and network diagrams (see Open items).

## 3. Background

### 3.1 What the diagram is built from

The Proxmox API returns one block per VM or container. These fields become the diagram:

| Field | Example | Becomes |
| --- | --- | --- |
| `node` | `pve` | Which host box the VM sits inside |
| `name`, `vmid` | `pfsense`, `100` | The guest's label |
| `type` | `qemu` (VM) or `lxc` (container) | `VM` or `CT` in the label |
| `status` | `running` / `stopped` | Colour |
| `template` | `0` / `1` | Templates drawn dashed |
| `maxmem` | `2147483648` | Memory, in **bytes** (divide by 1,073,741,824 for GB) |
| `maxcpu` | `2` | vCPU count |
| `tags` | `ubuntu01;qapp` | Shown in the label. Set in Proxmox, separated by `;` |

### 3.2 Standalone hosts vs a cluster

In a cluster, one query returns every guest on every host. Standalone hosts each know only their own guests and keep their own users, tokens and certificate authority. So the script queries each host separately, and Parts B to G are repeated on each host. `/cluster/resources` still works on a standalone host; it returns that host's guests only.

### 3.3 Host history

The install date of each host is the `notBefore` date of its CA (`/etc/pve/pve-root-ca.pem`), which is created at install and never regenerated.

| Host | Installed | Built on |
| --- | --- | --- |
| `pve` | October 2025 | First-build network and domain (`testlab.local`) |
| `pve02` | July 2026 | Earlier addressing and domain (`techworkslab.local`) |
| `pve03` | August 2026 | Current addressing and `techworks.local` |

This explains the certificate problems in Part G: the two older hosts still had their first-build address and domain in `/etc/hosts`, the DNS search domain and their node certificates.

## 4. Prerequisites

- Administrator access to each Proxmox web interface (`https://<pve-address>:8006`).
- A shell on the Ubuntu DevOps server as the account that will run the script (not root).
- Python 3.8 or later on the DevOps server (standard library only; nothing to install).
- The repository `techworks-enterprise-lab` cloned on the DevOps server.

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
- `pvesh get` prints a list of blocks, one per guest, containing the fields in section 3.1.

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
> Comment: Read-only account for lab diagram script
> leave Enabled ticked
> Add
```

If a password is requested, set a long random one and do not keep it. The account is used only through its API token.

**Expected result:** the Users list shows `diagram` with Realm `pve`, beside `root` with Realm `pam`.

**Why:** a script should never use the root password. A dedicated account can be audited, limited and removed without affecting anything else. The `pve` realm stores the user in Proxmox's own database; it is not a Linux account on the host.

**Automation equivalent:**

```
pveum user add diagram@pve --comment "Read-only account for lab diagram script"
```

> **Roadblock:** the user was first created as `digram` on `pve`. Proxmox cannot rename a user. **Fix:** select the row, Remove, and add it again with the correct name. Do this before creating a token, because the user name is part of the token ID. Check the selected row before clicking Remove: removing `root@pam` would lock you out of the web interface.

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
> Comment: Diagram script - read only
> Add
```

A window shows two values:

| Field | Example | Secret? |
| --- | --- | --- |
| Token ID | `diagram@pve!diagram-script` | No |
| Secret | `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` (36 characters) | **Yes. Shown only once** |

**Keep the window open** and store the secret straight away with Part E, step E1. It cannot be displayed again.

**Expected result:** the API Tokens list shows user `diagram@pve`, Token ID `diagram-script`, Privilege Separation `Yes`.

**Why:**

- **Privilege Separation** means the token's permissions are the **overlap** of the permissions given to the token and those given to its user. It is the stricter setting: the token can never do more than either one allows.
- A token can be revoked on its own without touching the user, and its use is identifiable in logs.
- Never paste the secret into a chat, a ticket or the repository. The repository's pre-commit hook does not recognise the Proxmox token format, so it would not catch one.

**Automation equivalent:**

```
pveum user token add diagram@pve diagram-script --privsep 1 --comment "Diagram script - read only"
```

This prints the secret once, in the same way.

## Part D: Give the user and the token read-only access

**Outcome:** the token can read everything on the host and change nothing.

Two permission rows are needed: one for the user and one for the token.

**Steps, row 1 (the user):**

```
Datacenter
> Permissions
> Add
> User Permission
> Path: /
> User: diagram@pve
> Role: PVEAuditor
> leave Propagate ticked
> Add
```

**Steps, row 2 (the token):**

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

**Expected result:** the Permissions list shows two rows on Path `/` with Role `PVEAuditor`: one for `diagram@pve` and one for `diagram@pve!diagram-script`. The Propagate column (`true`) may be off-screen on a narrow window.

**Why:**

- With Privilege Separation, the token gets the overlap of the user's and the token's permissions. If only one of them has a role, the overlap is empty and the token sees nothing.
- **`PVEAuditor`** is Proxmox's built-in read-only role: it can view hosts, guests, storage and settings, but cannot start, stop, change or delete anything.
- **Path `/`** is the top of the host's permission tree. **Propagate** applies the permission to everything below it, so guests created later are covered automatically.
- AWS comparison: the overlap works like an IAM **permissions boundary**. Effective access is what both the identity policy and the boundary allow.

**Automation equivalent:**

```
pveum acl modify / --users diagram@pve --roles PVEAuditor --propagate 1
pveum acl modify / --tokens 'diagram@pve!diagram-script' --roles PVEAuditor --propagate 1
```

Use single quotes around the token ID: in Bash, `!` inside double quotes triggers history expansion.

> **Roadblock:** on `pve`, only the token row was added at first. The API answered `200` with `"data": []`: no error, just nothing. Proxmox does not return `403` here; it leaves out every guest the token cannot see. **Fix:** add the user row (recorded at the time as "Part D2"). `pve02` and `pve03` got both rows from the start.

## Part E: Store the secret and test the API from the DevOps server

**Outcome:** each token secret is stored privately on the DevOps server, and a read over the network works.

All steps in this Part run on the Ubuntu DevOps server, as the account that will run the script. Check first:

```
hostname
whoami
```

**Expected result:** the DevOps server's name and the script account. Not a Proxmox host, not root.

### E1. Store the secret

```
mkdir -p ~/.config/techworks
chmod 700 ~/.config/techworks
( umask 077; read -rs PVE_SECRET; printf '%s' "$PVE_SECRET" > ~/.config/techworks/pve-diagram.token; unset PVE_SECRET )
wc -c ~/.config/techworks/pve-diagram.token
ls -l ~/.config/techworks/
```

When the cursor waits, paste the secret **once** and press Enter. Nothing is shown while you paste; that is expected. Use `pve02-diagram.token` and `pve03-diagram.token` for the other hosts.

**Expected result:** `wc` shows `36`. `ls` shows the file as `-rw-------`, owned by the script account.

**Why:**

- `read -rs` keeps the secret off the screen and out of shell history.
- `umask 077` means the file is created as `600` from the first moment.
- The parentheses run everything in a subshell, so the variable disappears afterwards.
- `~/.config/techworks` is outside the repository, so the secret can never be committed.

> **Roadblock:** the first file was 72 bytes: the secret had been pasted twice because `read -rs` gives no feedback. **Fix:** run the same command again and paste once. The `wc -c` check exists to catch this.

### E2. Test a read

```
printf 'Authorization: PVEAPIToken=%s=%s\n' 'diagram@pve!diagram-script' "$(cat ~/.config/techworks/pve-diagram.token)" \
  | curl -sk -H @- -o /dev/null -w '%{http_code}\n' "https://<pve-address>:8006/api2/json/cluster/resources?type=vm"
```

Then count the guests returned:

```
printf 'Authorization: PVEAPIToken=%s=%s\n' 'diagram@pve!diagram-script' "$(cat ~/.config/techworks/pve-diagram.token)" \
  | curl -sk -H @- "https://<pve-address>:8006/api2/json/cluster/resources?type=vm" \
  | python3 -m json.tool | grep -c '"vmid"'
```

**Expected result:** `200`, then a number that matches the guests shown under that host in the web interface. In this lab: `pve` 3, `pve02` 4, `pve03` 3.

**Why:**

- `printf` is a Bash builtin, so the secret never appears in the process list (`ps`), where other users could see it. `curl -H @-` reads the header from standard input.
- The token ID is in single quotes so `!` is not treated as history expansion.
- `-k` skips certificate checks. It is acceptable for this one test; Part G removes the need for it.
- Counting `vmid` lines keeps guest details off the screen.

| Result | Meaning | Check |
| --- | --- | --- |
| `200` and the right count | Working | None |
| `200` and `0` | Token accepted but can see nothing | Both permission rows in Part D |
| `401` | Token ID or secret rejected | Token name; secret file size (`wc -c`) |
| `403` | Signed in but not allowed | Part D |
| `000` | No HTTP reply at all | Section 8, "`000` / Connection refused" |

> **Roadblock:** the first test returned `000`, then `Connection refused`, although ping worked. The address used belonged to a different machine that answers ping but runs nothing on 8006. **Fix:** copy the address from the browser's address bar while the `pve` web interface is open. **Lesson:** ping uses ICMP; the API uses TCP 8006. A successful ping proves nothing about the port. "Refused after 0 ms" means something answered "nothing here"; a firewall drop would time out instead.

## Part F: The diagram script

**Outcome:** one command reads all three hosts and writes a Mermaid diagram.

### F1. Host list outside the repository

```
nano ~/.config/techworks/pve-hosts.ini
```

```
[pve]
address = <pve-address>
token_file = ~/.config/techworks/pve-diagram.token
ca_file = ~/.config/techworks/ca/pve-root-ca.pem

[pve02]
address = <pve02-address>
token_file = ~/.config/techworks/pve02-diagram.token
ca_file = ~/.config/techworks/ca/pve02-root-ca.pem

[pve03]
address = <pve03-address>
token_file = ~/.config/techworks/pve03-diagram.token
ca_file = ~/.config/techworks/ca/pve03-root-ca.pem
```

```
> Ctrl+O
> Enter
> Ctrl+X
```

```
chmod 600 ~/.config/techworks/pve-hosts.ini
```

The `ca_file` lines are added in Part G. Leave them out until the CA files exist.

**Expected result:** `ls -l ~/.config/techworks/` shows `pve-hosts.ini` and the three token files, all `-rw-------`.

**Why:** the script is given the path of this file, never the addresses. The repository copy of the script contains nothing environment-specific.

### F2. The script

The script is `tools/diagrams/pve_to_mermaid.py` in the repository. It uses only the Python standard library. For each host in the INI file it:

1. reads the token secret from `token_file` and warns if it is not 36 characters;
2. calls `/api2/json/cluster/resources?type=vm`, verifying TLS against `ca_file`;
3. writes one Mermaid `subgraph` per host and one node per guest (name, VM or CT, ID, status, tags);
4. replaces anything that looks like an IPv4 address with `[ip]`, including in error messages;
5. colours guests: green running, grey stopped, dashed blue template, red if the host could not be read.

It exits with code `1` if any host fails, so a scheduled job or CI pipeline can detect it.

| Option | Purpose |
| --- | --- |
| `-c <file>` | Use a different INI file (default `~/.config/techworks/pve-hosts.ini`) |
| `-o <file>` | Write the diagram to a file instead of the screen |
| `--insecure` | Skip certificate checks. For a first test only |
| `--timeout <s>` | Seconds to wait per host (default 10) |

### F3. Install and first run

From the repository root on the DevOps server:

```
cd ~/twhq/techworks-enterprise-lab
mkdir -p tools/diagrams
nano tools/diagrams/pve_to_mermaid.py
chmod 755 tools/diagrams/pve_to_mermaid.py
python3 -m py_compile tools/diagrams/pve_to_mermaid.py && echo OK
python3 tools/diagrams/pve_to_mermaid.py --insecure > /dev/null
```

**Expected result:** `OK`, then one line per host:

```
[pve] 3 guests
[pve02] 4 guests
[pve03] 3 guests
```

> **Roadblock:** `> /dev/dull` (a typo for `/dev/null`) gave `Permission denied`. Adding `sudo` does not help: the redirect is opened by your own shell, not by the command, and as root `~` points to `/root`, so the script cannot find the INI file. **Fix:** correct the typo; never run the script with `sudo`.

> **Roadblock:** running `python3 tools/diagrams/pve_to_mermaid.py` from the home folder gave `No such file or directory`. The path is relative to the repository root. **Fix:** `cd ~/twhq/techworks-enterprise-lab` first, or use the full path.

## Part G: Verify TLS instead of skipping it

**Outcome:** the script checks each host's certificate, and `--insecure` is no longer needed.

### G1. Copy each host's CA to the DevOps server

```
mkdir -p ~/.config/techworks/ca
scp root@<pve-address>:/etc/pve/pve-root-ca.pem   ~/.config/techworks/ca/pve-root-ca.pem
scp root@<pve02-address>:/etc/pve/pve-root-ca.pem ~/.config/techworks/ca/pve02-root-ca.pem
scp root@<pve03-address>:/etc/pve/pve-root-ca.pem ~/.config/techworks/ca/pve03-root-ca.pem
for h in pve pve02 pve03; do openssl x509 -in ~/.config/techworks/ca/$h-root-ca.pem -noout -subject; done
```

If root SSH is disabled, open each host's **Shell**, run `cat /etc/pve/pve-root-ca.pem`, and paste everything from `-----BEGIN CERTIFICATE-----` to `-----END CERTIFICATE-----` into the matching file with `nano`.

**Expected result:** three files of about 2 KB, and three `subject=` lines each with `CN = Proxmox Virtual Environment`, `O = PVE Cluster Manager CA` and a **different** `OU` value. Identical lines mean the same file was copied twice.

**Why:** the CA is a public certificate, not a secret. Each standalone host created its own at install time.

### G2. Point the script at the CAs and test

Add the `ca_file` lines to `pve-hosts.ini` (Part F1), then:

```
cd ~/twhq/techworks-enterprise-lab
python3 tools/diagrams/pve_to_mermaid.py > /dev/null
```

**Expected result:** the same three guest counts as in F3, with no `--insecure`.

If a host shows `CERTIFICATE_VERIFY_FAILED ... IP address mismatch`, its node certificate does not list the address you connect with. Do G3 on that host.

### G3. Correct the host's name and address, then rebuild its certificate

**Check what the certificate covers** (manual):

```
<host>
> System
> Certificates
> pve-ssl.pem
> View Certificate
```

Look at **Subject Alternative Names**. Or, from the DevOps server:

```
echo | openssl s_client -connect <pve-address>:8006 2>/dev/null | openssl x509 -noout -ext subjectAltName
```

**Check the host's own settings.** In the host's Shell (check `hostname` first):

```
hostname
hostname --ip-address
grep "$(hostname)" /etc/hosts
grep search /etc/resolv.conf
ip -br addr
```

**Expected result on a correct host:** the address on `vmbr0`, the `hostname --ip-address` result and the `/etc/hosts` line all agree; the `/etc/hosts` line reads `<address> <host>.techworks.local <host>`; and the search line reads `search techworks.local`.

**Fix, in this order:**

1. Back up `/etc/hosts`:

   ```
   cp /etc/hosts /root/hosts.bak-$(date +%F)
   ```

2. Correct the hosts entry:

   ```
   <host>
   > System
   > Hosts
   ```

   Change the host's line to `<current-address> <host>.techworks.local <host>`.

   ```
   > Save
   ```

3. Correct the DNS search domain:

   ```
   <host>
   > System
   > DNS
   > Edit
   > Search domain: techworks.local
   > leave the DNS servers as they are
   > OK
   ```

   Confirm in the Shell with `grep search /etc/resolv.conf` before going on.

4. Rebuild the node certificate, in the Shell:

   ```
   pvecm updatecerts --force
   openssl x509 -in /etc/pve/local/pve-ssl.pem -noout -ext subjectAltName | grep -o 'DNS:[^,]*\.local'
   ```

   **Expected result:** `DNS:<host>.techworks.local`.

5. Restart the web and API service from the GUI:

   ```
   <host>
   > System
   > pveproxy
   > Restart
   ```

6. Refresh the browser and accept the certificate warning (the certificate is new). The CA has not changed, so the CA file on the DevOps server stays valid.

**Why:**

- `pvecm updatecerts --force` rebuilds `pve-ssl.pem`, the web and API certificate. It takes the **IP address** from `/etc/hosts` and the **DNS name** from the hostname plus the search domain in `/etc/resolv.conf`. Both must be right before you run it.
- At boot, Proxmox only creates a node certificate if it is missing or expired. **Rebooting does not fix a wrong certificate.** Only `--force` does.
- `pveproxy` keeps serving the old certificate until it is restarted.
- Pointing the hosts' DNS servers at the domain controller is a separate change (see Open items). If the DC is down, name lookups on the host would fail, and an A record for the host should exist in AD DNS first.

**Automation equivalent** (not run in this lab):

```
sed -i 's/^.*\s<host>\.<old-domain>\s<host>$/<current-address> <host>.techworks.local <host>/' /etc/hosts
sed -i 's/^search .*/search techworks.local/' /etc/resolv.conf
pvecm updatecerts --force
systemctl restart pveproxy
```

### G4. Check all three hosts

From the DevOps server. This reads the addresses from the INI file, so nothing is typed or shown:

```
for a in $(awk -F' = ' '/^address/ {print $2}' ~/.config/techworks/pve-hosts.ini); do
  echo | openssl s_client -connect "$a":8006 2>/dev/null | openssl x509 -noout -ext subjectAltName | grep -o 'DNS:[^,]*\.local'
done
```

**Expected result:**

```
DNS:pve.techworks.local
DNS:pve02.techworks.local
DNS:pve03.techworks.local
```

> **Roadblock:** on `pve` and `pve02`, the node certificates contained first-build addresses. `pve`'s `/etc/hosts` pointed to yet another old address, so its certificate was older than its hosts entry. `pve03` was built later and was already correct.

> **Roadblock:** step G3's host checks were first run on the DevOps server by mistake (the `127.0.1.1 <devops-server>` line came back). The `hostname` check at the top caught it.

> **Roadblock:** restarting `pveproxy` from the web Shell cut the Shell off with `Connection closed (Code: 1006)`. Expected, because the web Shell runs through `pveproxy`. The commands had already finished. **Fix:** restart from System > pveproxy, or over SSH.

> **Roadblock:** after fixing `/etc/hosts`, the certificate still showed `pve.testlab.local` (and `pve02.techworkslab.local` on `pve02`). The DNS name comes from the search domain, not `/etc/hosts`. **Fix:** step 3, then `--force` again.

> **Roadblock:** `pve02` was rebooted twice after the search domain was fixed, but kept serving the old name. `pve-ssl.pem` still had its earlier timestamp. **Fix:** `pvecm updatecerts --force` and a `pveproxy` restart. Rebooting never rebuilds an existing certificate.

> **Roadblock:** a `for` loop was written with commas between the addresses (`for a in x,y,z`), so it ran once with one invalid host, and `openssl` reported `Could not read certificate from <stdin>`. Bash lists are space-separated. The INI-reading loop above avoids typing addresses at all.

## Part H: Write the diagram

```
cd ~/twhq/techworks-enterprise-lab
python3 tools/diagrams/pve_to_mermaid.py -o docs/diagrams/proxmox-guests.mmd
echo "exit code: $?"
```

**Expected result:** three status lines, `Wrote docs/diagrams/proxmox-guests.mmd`, and `exit code: 0`. The file has three `subgraph` blocks with 3, 4 and 3 guest lines.

Before committing, read the file for anything that should not be public: addresses (the script replaces these) and guest names or tags that reveal personal details (the script cannot judge these). To view it as a picture, paste it into mermaid.live.

---

## 6. Verification

On each Proxmox host shell:

```
pveum user list
pveum user token list diagram@pve
pveum acl list
pveum user token permissions diagram@pve diagram-script
```

**Expected result:**

- `diagram@pve` is listed as enabled, and its token list contains `diagram-script` with `privsep` `1`.
- `pveum acl list` has two rows on `/` with role `PVEAuditor` and propagate `1`: one of type `user` for `diagram@pve`, one of type `token` for `diagram@pve!diagram-script`.
- The token permissions list shows only `.Audit` privileges (for example `VM.Audit`, `Sys.Audit`, `Datastore.Audit`), and nothing ending in `.Allocate`, `.Modify` or `.PowerMgmt`.

On the DevOps server:

```
ls -l ~/.config/techworks/ ~/.config/techworks/ca/
cd ~/twhq/techworks-enterprise-lab && python3 tools/diagrams/pve_to_mermaid.py > /dev/null; echo "exit code: $?"
```

**Expected result:** token files and `pve-hosts.ini` are `-rw-------`; three CA files exist; a guest count for each host and `exit code: 0`.

## 7. Rollback

| To undo | Do this |
| --- | --- |
| The permissions | Datacenter > Permissions > select each `/` row for `diagram@pve` and `diagram@pve!diagram-script` > Remove. Or `pveum acl delete / --users diagram@pve --roles PVEAuditor` and `pveum acl delete / --tokens 'diagram@pve!diagram-script' --roles PVEAuditor` |
| The token | Datacenter > Permissions > API Tokens > select `diagram@pve!diagram-script` > Remove. Or `pveum user token remove diagram@pve diagram-script` |
| The user | Datacenter > Permissions > Users > select `diagram` > Remove. Or `pveum user delete diagram@pve`. Removing the user also removes its tokens |
| The stored secrets, host list and CAs | On the DevOps server: `rm -r ~/.config/techworks` |
| A hosts-file change | `cp /root/hosts.bak-<date> /etc/hosts` on that host, then `pvecm updatecerts --force` and restart `pveproxy` |
| The script and diagram | `git rm tools/diagrams/pve_to_mermaid.py docs/diagrams/proxmox-guests.mmd` |

## 8. Troubleshooting

| Symptom | Likely cause | Check | Fix |
| --- | --- | --- | --- |
| `pvecm status`: "Corosync config ... does not exist" | The host is standalone | Expected on standalone hosts | None. Query each host separately |
| API returns `200` with `"data": []` | With Privilege Separation, only the token or only the user has the role | `pveum acl list` | Add the missing row (Part D) |
| API returns `401` | Wrong token ID or secret | `wc -c` on the secret file (should be 36) | Re-store the secret (E1) or re-create the token |
| API returns `403` | No permission on the path | `pveum acl list` | Part D |
| `000` / `Connection refused` | Wrong address, or nothing listening on 8006 | Browser address bar of the working web UI; `nc -zv -w 5 <address> 8006`; on the host `ss -tlnp \| grep 8006` | Use the right address; `systemctl status pveproxy` |
| `000` / timed out | Traffic dropped by a firewall | pfSense rules, Proxmox firewall | Allow TCP 8006 from the DevOps server |
| Secret file is 72 bytes | Pasted twice into `read -rs` | `wc -c` | Run E1 again and paste once |
| `bash: !diagram: event not found` | `!` in double quotes triggers history expansion | n/a | Put the token ID in single quotes |
| `CERTIFICATE_VERIFY_FAILED ... IP address mismatch` | Node certificate built with an old address | Subject Alternative Names (G3) | Fix `/etc/hosts`, `pvecm updatecerts --force`, restart `pveproxy` |
| Certificate has the old domain name | Search domain still old | `grep search /etc/resolv.conf` | Fix the search domain, then `--force` again |
| Certificate unchanged after a reboot | Boot only creates missing or expired certificates | `ls -l /etc/pve/local/pve-ssl.pem` timestamp | `pvecm updatecerts --force` |
| Certificate rebuilt but old one still served | `pveproxy` not restarted, or a custom `pveproxy-ssl.pem` exists | `ls /etc/pve/local/pveproxy-ssl.pem` | Restart `pveproxy`; review the custom certificate |
| Web Shell shows `Connection closed (Code: 1006)` | `pveproxy` was restarted from inside it | n/a | Expected. Refresh the page |
| `Could not read certificate from <stdin>` | `openssl s_client` could not connect (bad address or loop syntax) | The address list | Space-separate loop items; use the INI-reading loop |
| `Permission denied` writing to `/dev/...` | Typo in the redirect target | The command line | Use `/dev/null`; do not add `sudo` |
| `can't open file ... pve_to_mermaid.py` | Run from outside the repository root | `pwd` | `cd ~/twhq/techworks-enterprise-lab` |
| A host shows `could not read this host` in the diagram | That host failed | The script's error line for it | Use the rows above |

## 9. Key lessons

> ### A script gets its own account and a read-only token, never the root password.

- **Standalone hosts each need their own set-up.** Users, tokens and CAs are per host unless the hosts are clustered.
- **With Privilege Separation, both the user and the token need the role.** The token gets the overlap, like an AWS permissions boundary, and an empty overlap returns empty data rather than an error.
- **A secret shown once must be captured immediately**, stored outside the repository, and kept out of history and the process list.
- **Ping proves the host is up, not that the service is reachable.** Test the actual port.
- **Fix the source, not the symptom.** `--insecure` would have hidden stale addresses and domain names that would have caused problems elsewhere.
- **A host's name, address and search domain must agree before you rebuild its certificate**, and rebooting does not rebuild it.
- **Run `hostname` before any command that changes a machine.**

## 10. Command reference

| Task | Command |
| --- | --- |
| Is this host in a cluster? | `pvecm status` |
| List guests on this host | `pvesh get /cluster/resources --type vm --output-format json-pretty` |
| Create a Proxmox user | `pveum user add diagram@pve --comment "<text>"` |
| Create an API token with privilege separation | `pveum user token add diagram@pve diagram-script --privsep 1` |
| Give the user read-only access | `pveum acl modify / --users diagram@pve --roles PVEAuditor --propagate 1` |
| Give the token read-only access | `pveum acl modify / --tokens 'diagram@pve!diagram-script' --roles PVEAuditor --propagate 1` |
| List all permissions on the host | `pveum acl list` |
| Show what a token can actually do | `pveum user token permissions diagram@pve diagram-script` |
| Store a secret without echo or history | `( umask 077; read -rs S; printf '%s' "$S" > <file>; unset S )` |
| Check a file's size in bytes | `wc -c <file>` |
| Test the API, secret kept out of `ps` | `printf 'Authorization: PVEAPIToken=%s=%s\n' '<token-id>' "$(cat <file>)" \| curl -sk -H @- <url>` |
| Test a TCP port | `nc -zv -w 5 <address> 8006` |
| See what is listening on 8006 | `ss -tlnp \| grep 8006` |
| Show a host's addresses | `ip -br addr` |
| Show a CA's subject or start date | `openssl x509 -in <ca.pem> -noout -subject` / `-startdate` |
| Show a served certificate's names | `echo \| openssl s_client -connect <address>:8006 2>/dev/null \| openssl x509 -noout -ext subjectAltName` |
| Rebuild a node certificate | `pvecm updatecerts --force` |
| Restart the web and API service | `systemctl restart pveproxy` (or System > pveproxy > Restart) |
| Run the script, counts only | `python3 tools/diagrams/pve_to_mermaid.py > /dev/null` |
| Write the diagram | `python3 tools/diagrams/pve_to_mermaid.py -o docs/diagrams/proxmox-guests.mmd` |

## 11. Design notes

- **Per-host tokens.** Three standalone hosts mean three users, three tokens and three CAs. Clustering, or Proxmox Datacenter Manager (stable 1.0 released December 2025), would centralise management; either would be a separate phase.
- **Secrets and addresses stay outside the repository.** The script reads `~/.config/techworks/pve-hosts.ini` (mode 600) and the token files (mode 600). The repository holds only the script and the generated diagram.
- **Verified TLS.** The script trusts each host's own CA instead of disabling checks. `--insecure` remains for first-time tests only.
- **Inventory vs architecture.** The generated Mermaid diagram is the current-state inventory. The target architecture is drawn by hand (UniFi topology view for the network, draw.io for the design) and stored alongside it. A `.drawio` file is plain XML and can be committed.
- **Exit codes.** The script returns `1` if any host fails, so it can later run on a schedule or in a CI pipeline and alert on failure.

## 12. Open items

- [x] Parts A to D on all three hosts (both permission rows).
- [x] Part E: secrets stored and API read tested from the DevOps server (2026-11-17).
- [x] Part F: host list and diagram script (2026-11-18).
- [x] Part G: CAs trusted; certificates on `pve` and `pve02` corrected to current addresses and `techworks.local` (2026-11-18).
- [x] Part H: `docs/diagrams/proxmox-guests.mmd` generated (2026-11-18).
- [ ] Active Directory diagram: OUs and GPO links.
- [ ] Network diagram: UniFi topology export plus a draw.io target architecture.
- [ ] Add the diagrams to the README.
- [ ] Point the Proxmox hosts' DNS servers at the domain controller, after A records exist in AD DNS.
- [ ] Trust the three Proxmox CAs in the desktop browser.
- [ ] Review the two stopped Red Hat VMs (one on `pve02`, one on `pve03`) before SAM-ADM-004.
- [ ] Confirm whether `DC01` being stopped is intentional (only `DC02` was running).

## Document history

| Date | Change | By |
| --- | --- | --- |
| 2026-11-17 | First issue: Parts A to C | Techworks |
| 2026-11-17 | Added Part D (read-only permission), verification, rollback and commands | Techworks |
| 2026-11-19 | Part D corrected to two rows (user and token). Added Parts E to H, host history, roadblocks, troubleshooting and commands. Proxmox source complete on all three hosts | Techworks |
