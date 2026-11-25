# SAM-ADM-004: Red Hat foundation

| Field | Value |
| --- | --- |
| Document ID | SAM-ADM-004 |
| Title | Building a Red Hat Enterprise Linux 10 server and joining it to Active Directory |
| Owner / Author | Techworks |
| Status | Complete. Built and verified, including a reboot test, 2026-11-24 to 2026-11-25. Follow-up items listed under Open items |
| Last updated | 2026-11-25 |
| Depends on | Domain controllers DC-01 and DC-02 (`techworks.local`), each with a static address and DNS pointing at the other |

**Search keywords:** `hostnamectl`, `set-hostname`, `/etc/redhat-release`, `Permission denied`, `timedatectl`, `System Not Registered`, `local repository`, `BaseOS`, `AppStream`, `RHCSA`, `EX200`, `realm join`, `sssd`, `iso9660`, `/etc/fstab`, `findmnt --verify`, `rhel-dvd.repo`, `gpgcheck`, `Unable to read consumer identity`, `subscription-manager.conf`, `nmtui`, `nmcli connection modify`, `proto static`, `resolv.conf`, `getent hosts`, `in-addr.arpa`, `reverse lookup zone`, `PTR`, `ipconfig /registerdns`, `Add-DnsServerPrimaryZone`, `realm discover`, `Join is successful`, `kerberos-member`, `use_fully_qualified_names`, `getent ahostsv4`, `realm permit`, `allow-permitted-logins`, `sss_cache`, `visudo`, `sudoers.d`, `mode 0440`, `SamAccountName`, `break-glass`, `Cockpit`, `9090`, `profile.d`, `login banner`, `ssh -l`, `:set paste`

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
- **Where** at the top of each step says which machine to run it on.
- Placeholders: `<rhel-address>`, `<dc01-address>`, `<dc02-address>`, `<gateway>`, `<network>` (the lab subnet), `<reversed-network>` (its first three octets reversed, as in reverse zone names), `<local-admin>` (the local break-glass account created at install), `<domain-admin>` (a named AD admin account). Real values never go in this document or the repository.

---

## 1. Purpose and outcome

Build a Red Hat Enterprise Linux (RHEL) 10 server, give it a stable identity on the network, and join it to the `techworks.local` Active Directory domain so domain users can sign in and AD groups control who gets administrator rights.

The build follows the RHCSA (EX200) approach: software comes from a local repository on the installation ISO rather than from a Red Hat subscription.

**You know it works when:**

- `hostnamectl` shows `Static hostname: twredhat.techworks.local`.
- `sudo dnf repolist` lists the two DVD repositories, and packages install from them with signature checks.
- `ip route` shows `proto static`, and `getent hosts dc-02.techworks.local` resolves through the DCs.
- `realm list` shows `configured: kerberos-member`, and `id amy.fin@techworks.local` lists her AD groups.
- Only members of `dl_twredhat_admins` (through `gg_linux_admins`) can sign in, and they get `sudo`; `amy.fin` is refused.
- Signing in shows the server-facts banner, and all of the above survives a reboot.

## 2. Scope and environment

| Item | Value |
| --- | --- |
| Server | `twredhat.techworks.local`, VM 500 on Proxmox host `pve02` |
| OS | Red Hat Enterprise Linux 10.3 (Coughlan), Server with GUI |
| Hardware (virtual) | 2 vCPU, 8 GB RAM, 100 GB disk, network card `ens18` |
| Firmware / machine | SeaBIOS, i440FX (Proxmox defaults) |
| Subscription | None, by design (local ISO repository) |
| Domain | `techworks.local`. DC-02 is the main DC; DC-01 is the second |

## 3. Prerequisites

- Console access to the VM (Proxmox > `pve02` > 500 > Console), or SSH once it is set up.
- An account on the VM with `sudo` rights.
- The RHEL 10.3 DVD ISO on `pve02`, with its SHA-256 matching Red Hat's published value.

---

## Part A: Record the server's facts and set its name

**Outcome:** the server has a final, lowercase, fully qualified hostname, and its facts are recorded.

**Where:** a terminal on the RHEL VM.

### A1. Set the hostname

```
nmcli general hostname
sudo hostnamectl set-hostname twredhat.techworks.local
hostnamectl
```

**Expected result:** `Static hostname: twredhat.techworks.local`, `Chassis: vm`, `Virtualization: kvm`, `Operating System: Red Hat Enterprise Linux 10.3 (Coughlan)`.

The terminal prompt keeps showing `localhost` until a new terminal is opened. That is normal: the prompt is set when the shell starts.

**Why:**

- The domain join in a later Part turns the hostname into an AD computer account and Kerberos names. Names should be final before anything depends on them.
- Lowercase is the safe habit on Linux: DNS ignores case, but Kerberos and some tools do not.
- `hostnamectl set-hostname` writes `/etc/hostname`, so the name survives a reboot.

> **Roadblock:** the hostname was first set with capitals (`TWRedHat.techworks.local`) using `nmcli general hostname`. It worked, but was changed to lowercase before the domain join, for the reason above.

### A2. Record the facts

```
cat /etc/redhat-release
timedatectl
nproc; free -h; lsblk
ip -br addr
nmcli -g ipv4.method,ipv4.dns connection show "$(nmcli -g NAME connection show --active | head -1)"
```

**Expected result and recorded values:**

| Fact | Command | Value in this lab |
| --- | --- | --- |
| Release | `cat /etc/redhat-release` | Red Hat Enterprise Linux release 10.3 (Coughlan) |
| Time zone | `timedatectl` | America/New_York |
| Clock synchronised | `timedatectl` | `yes`, NTP service `active` |
| vCPU / RAM / disk | `nproc`, `free -h`, `lsblk` | 2 / 8 GB / 100 GB |
| Network card | `ip -br addr` | `ens18` |
| Address method | `ip a` shows `dynamic`; `nmcli` shows `auto` | DHCP (made static in Part C) |
| VM or physical | `hostnamectl` | VM (also: MAC prefix `bc:24:11` is Proxmox's) |
| Subscription | Desktop notice "System Not Registered" | Not registered, by design |

**Why:** these facts go into the server's record and decide the next steps. Time matters most for the domain join: Kerberos rejects sign-ins when clocks differ by more than 5 minutes.

> **Roadblock:** `/etc/redhat-release` typed on its own gave `Permission denied`, and `sudo /etc/redhat-release` gave `command not found`. It is a text file, not a program: the shell tried to run it. **Fix:** read it with `cat /etc/redhat-release`. `sudo` does not help, because the problem is not permission to read but that the file is not executable.

## Part B: Local repository from the installation ISO

**Outcome:** `dnf` installs signed packages from the RHEL DVD, with no subscription. This is the method used in the RHCSA exam.

### B1. Check the ISO on the Proxmox host

**Where:** the `pve02` host Shell (Datacenter > `pve02` > Shell), not the VM.

```
hostname
ls -l /var/lib/vz/template/iso/
sha256sum /var/lib/vz/template/iso/rhel-10.3-x86_64-dvd.iso
sha256sum /var/lib/vz/template/iso/rhel-10.3-x86_64-dvd.iso
```

**Expected result:** `pve02`; the ISO at 11,717,574,656 bytes; two identical hashes that match the SHA-256 published by Red Hat for the RHEL 10.3 Binary DVD (begins `e75131f8`).

**Why:** a damaged ISO produces checksum errors when installing packages. Hashing twice also proves the host reads data consistently: on `pve03`, two runs on the same file gave different results, which led to the failing RAM module.

> **Roadblock:** `ls -l /var/lig/vz/template/iso` gave `No such file or directory`. Typo for `/var/lib`. "No such file or directory" usually means check the spelling of the path first.

### B2. Attach the ISO to the VM

**Where:** Proxmox web interface. The VM can stay running.

```
pve02
> 500 (RedHat)
> Hardware
> CD/DVD Drive (ide2)
> Edit
> Use CD/DVD disc image file (iso)
> Storage: local
> ISO image: rhel-10.3-x86_64-dvd.iso
> OK
```

If there is no CD/DVD Drive row: Add > CD/DVD Drive, same settings.

Then, **on the VM**:

```
lsblk
sudo blkid /dev/sr0
```

**Expected result:** `sr0` of about 11G, type `rom`; `LABEL="RHEL-10-3-BaseOS-x86_64"`, `TYPE="iso9660"`.

### B3. Mount the DVD permanently at /repo

**Where:** the VM.

```
sudo cp /etc/fstab /etc/fstab.bak-$(date +%F)
sudo mkdir -p /repo
sudo vim /etc/fstab
```

Add at the end:

```
/dev/sr0   /repo   iso9660   ro,nofail   0 0
```

Test before any reboot:

```
sudo systemctl daemon-reload
sudo findmnt --verify
sudo mount -a
ls /repo
```

**Expected result:** `Success, no errors or warnings detected`; `mount -a` prints nothing; `/repo` contains `AppStream`, `BaseOS`, `EFI`, `images`, `media.repo` and the `RPM-GPG-KEY-redhat-*` files.

| Field | Value | Meaning |
| --- | --- | --- |
| 1 | `/dev/sr0` | The DVD drive |
| 2 | `/repo` | Mount point |
| 3 | `iso9660` | DVD filesystem type (as shown by `blkid`) |
| 4 | `ro,nofail` | Read-only; **boot continues if the disc is missing** |
| 5, 6 | `0 0` | No dump, no boot-time check |

**Why:** a mistake in `/etc/fstab` can stop the server booting (emergency mode). Back up, use `nofail`, and test with `findmnt --verify` and `mount -a` before rebooting.

> **Roadblock:** `sudo cp /etc/fstab /etc/fstab/.bak-<date>` gave `Not a directory`. The extra `/` made `cp` treat `fstab` as a folder. **Fix:** `/etc/fstab.bak-<date>`.

> **Roadblock:** `findmnt --verifiy` gave `unrecognized option`. Typo for `--verify`.

### B4. Define the repositories

**Where:** the VM.

```
sudo vim /etc/yum.repos.d/rhel-dvd.repo
```

```
[rhel-dvd-baseos]
name=RHEL 10.3 DVD - BaseOS
baseurl=file:///repo/BaseOS
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release

[rhel-dvd-appstream]
name=RHEL 10.3 DVD - AppStream
baseurl=file:///repo/AppStream
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
```

```
sudo dnf clean all
sudo dnf repolist
```

**Expected result:** `rhel-dvd-appstream` and `rhel-dvd-baseos` listed.

**Why:**

- RHEL splits software into **BaseOS** (core OS) and **AppStream** (applications and tools); many packages need both.
- `baseurl=file:///repo/BaseOS` is `file://` plus the path `/repo/BaseOS`: three slashes.
- **`gpgcheck=1`** verifies Red Hat's signature on every package. Keep it on.

**Automation equivalent (exam shortcut):** `sudo dnf config-manager --add-repo file:///repo/BaseOS` (and the same for AppStream). It creates the section without `gpgcheck`/`gpgkey`; add those by hand.

### B5. Silence the "not registered" message

The `subscription-manager` plugin for `dnf` prints "Unable to read consumer identity" and "This system is not registered" on every command. Harmless, but noisy.

```
sudo cp /etc/dnf/plugins/subscription-manager.conf /etc/dnf/plugins/subscription-manager.conf.bak
sudo vim /etc/dnf/plugins/subscription-manager.conf
```

Change `enabled=1` to `enabled=0`.

**Expected result:** the next `dnf` command prints no registration messages.

### B6. Prove it: install the domain-join packages

```
sudo dnf install realmd sssd adcli oddjob oddjob-mkhomedir samba-common-tools krb5-workstation
```

**Expected result:** already-installed packages are skipped; the rest are listed with Repository `rhel-dvd-baseos` or `rhel-dvd-appstream`; `Complete!`. In this lab, 9 packages were installed, including `krb5-workstation`, `samba-common-tools`, `oddjob` and `oddjob-mkhomedir`.

| Package | Role |
| --- | --- |
| `realmd` | Discovers and joins the domain |
| `sssd` | AD sign-in and offline cache |
| `adcli` | Creates the computer account in AD |
| `oddjob`, `oddjob-mkhomedir` | Create a home folder at first sign-in |
| `samba-common-tools` | Windows-compatible tools used by `realmd` |
| `krb5-workstation` | `kinit`, `klist` for testing Kerberos |

> **Roadblock:** the first tests (`realmd`, `tree`, `mtr`) proved nothing: all were already installed by the Server with GUI install (`Repository: @System`, `From repo: anaconda`). **Fix:** install something genuinely missing; the domain-join packages served both purposes.

`dbus-daemon: no process found` during installation is harmless: RHEL 10 uses `dbus-broker`, and the package script looked for the older daemon.

## Part C: Static address and DNS pointing at the DCs

**Outcome:** the server keeps the same address permanently, outside the DHCP pool, and uses the domain controllers for DNS.

**Address plan** (see RB-pfSense-022): `.1`–`.49` network gear; `.50`–`.99` hypervisors, DCs and servers (static on the device); `.100`–`.199` DHCP pool; `.200`–`.254` printers, reservations and spare. This server takes `<rhel-address>` in the server block.

**Where:** the VM's console, not SSH. Changing the address cuts an SSH session.

### C1. Check the address is free

```
ping -c 2 <rhel-address>
nmcli connection show
ip route | grep default
```

**Expected result:** the ping fails (`Destination Host Unreachable` or `100% packet loss`); one connection, `ens18`; `default via <gateway> ... proto dhcp`.

Also check the address is not in the address plan spreadsheet or pfSense's Status > DHCP Leases. Some devices ignore ping, so a failed ping alone does not prove an address is free.

### C2. Set the address with nmtui

```
sudo nmtui
```

```
> Edit a connection
> ens18
> Edit...
> IPv4 CONFIGURATION: <Automatic>  → Manual
> <Show>
> Addresses: Add...  → <rhel-address>/24
> Gateway: <gateway>
> DNS servers: Add...  → <dc02-address>
> DNS servers: Add...  → <dc01-address>
> Search domains: Add...  → techworks.local
> leave [X] Automatically connect ticked (near the bottom)
> leave [ ] Require IPv4 addressing unticked
> OK
> Back
> Activate a connection
> ens18  → Deactivate
> ens18  → Activate
> Back
> Quit
```

| Setting | Value | Why |
| --- | --- | --- |
| Address | `<rhel-address>/24` | `/24` = `255.255.255.0` |
| Gateway | `<gateway>` | The Netgate |
| DNS 1 / 2 | DC-02, then DC-01 | AD is found through AD's own DNS; DC-01 is the fallback |
| Search domain | `techworks.local` | Short names such as `dc-02` resolve |
| Automatically connect | Ticked | The network comes up at boot |
| Deactivate / Activate | | Saving does not apply the change; reactivating does |

**Why "nm":** `nmcli` (NetworkManager Command-Line Interface), `nmtui` (NetworkManager Text User Interface) and GNOME Settings all edit the same NetworkManager profile, stored in `/etc/NetworkManager/system-connections/`.

**Automation equivalent:**

```
sudo nmcli connection modify ens18 ipv4.method manual ipv4.addresses <rhel-address>/24 ipv4.gateway <gateway> ipv4.dns "<dc02-address> <dc01-address>" ipv4.dns-search techworks.local
sudo nmcli connection up ens18
```

### C3. Verify

```
ip -br addr show ens18
ip route | grep default
cat /etc/resolv.conf
ping -c 2 <dc02-address>
getent hosts dc-02.techworks.local
```

**Expected result:**

- `ens18 UP <rhel-address>/24`, and the old DHCP address is gone.
- `default via <gateway> ... proto static` (it was `proto dhcp`).
- `/etc/resolv.conf`: `search techworks.local`, then the two DC addresses as `nameserver` lines.
- Two ping replies from DC-02. `ttl=128` is typical of Windows; Linux usually answers with 64.
- `getent` prints DC-02's address and `dc-02.techworks.local`: the server is resolving names through AD DNS.

Update the address plan spreadsheet: `twredhat` moved from its DHCP address to `<rhel-address>`. The old lease expires on its own.

## Part D: Reverse lookup zone and PTR records

**Outcome:** addresses on the lab network can be looked up by name *and* names by address.

| Record | Direction | Example | Used by |
| --- | --- | --- | --- |
| **A** | Name → address (forward) | `dc-02.techworks.local` → `<dc02-address>` | `ping <name>`, almost everything |
| **PTR** | Address → name (reverse) | `<dc02-address>` → `dc-02.techworks.local` | `nslookup <address>`, `ping -a`, logs, Kerberos and some services |

PTR records live in a separate reverse lookup zone. Before this Part, the domain had none.

### D1. Create the reverse lookup zone

**Where:** DC-02, signed in as a domain administrator.

```
Server Manager
> Tools
> DNS
> expand DC-02
> right-click Reverse Lookup Zones
> New Zone...
> Next
> Primary zone
> leave "Store the zone in Active Directory" ticked
> Next
> To all DNS servers running on domain controllers in this domain: techworks.local
> Next
> IPv4 Reverse Lookup Zone
> Next
> Network ID: <first three octets of the lab network>
> Next
> Allow only secure dynamic updates (recommended for Active Directory)
> Next
> Finish
```

**Expected result:** `<reversed-network>.in-addr.arpa`, type Active Directory-Integrated Primary, status Running. A few minutes later the same zone appears on DC-01 (DNS Manager > right-click DNS > Connect to DNS Server > DC-01).

| Choice | Why |
| --- | --- |
| Stored in AD, all DNS servers in the domain | Replicates with AD; both DCs answer for it |
| Network ID (first three octets) | Covers the whole `/24`. Reverse zones are written backwards |
| Secure dynamic updates | Only domain members can register or update their own records |

**Automation equivalent** (PowerShell on a DC):

```
Add-DnsServerPrimaryZone -NetworkId "<network>/24" -ReplicationScope Domain -DynamicUpdate Secure
```

### D2. Register the DCs' PTR records and test both directions

**Where:** each DC, in an elevated Command Prompt or PowerShell.

```
hostname
ipconfig /registerdns
```

After a minute, refresh the zone (F5) and test:

```
nslookup dc-01.techworks.local
nslookup <dc01-address>
nslookup <dc02-address>
```

**Expected result:** two Pointer (PTR) entries in the zone; forward and reverse lookups both return `DC-01.techworks.local` and `DC-02.techworks.local`.

**Why `ipconfig /registerdns`:** Windows registers its records at boot and every 24 hours. Before the zone existed, PTR registrations had nowhere to go; this makes Windows retry immediately. For other Windows machines: `Invoke-Command -ComputerName <pc1>,<pc2> -ScriptBlock { ipconfig /registerdns }`.

**`twredhat`'s A and PTR records** are created by the domain join in Part E: `sssd` registers them with the server's own computer account, so it can update them later. Records created by hand belong to the admin who created them, and the server cannot update them afterwards.

**Manual alternative** (if a non-domain device needs records): DNS Manager > Forward Lookup Zones > `techworks.local` > right-click > New Host (A or AAAA)... > Name, IP address > tick **Create associated pointer (PTR) record** > Add Host.

> **Roadblock (theory):** "Without PTR records you can't ping hostnames" is a common misunderstanding. `ping <name>` uses the **A** record. `twredhat` could not be pinged by name because it had no A record yet: it was not joined to the domain.

> **Roadblock:** `dig` is not a Windows command. On Windows use `nslookup` or `Resolve-DnsName`; `dig` is on Linux (package `bind-utils`).

**Ping errors tell you the layer:**

| Ping says | Meaning |
| --- | --- |
| `Ping request could not find host` | Name did not resolve: DNS |
| `Request timed out` | Name resolved, no reply: firewall or device off |
| `Reply from` | Working |

## Part E: Join the domain

**Outcome:** `twredhat` is a member of `techworks.local`; AD users and groups are visible on the server; its DNS records exist.

**Where:** the RHEL VM terminal, unless stated.

### E1. Discover the domain (read-only)

```
realm discover techworks.local
```

**Expected result:** `type: kerberos`, `realm-name: TECHWORKS.LOCAL`, `configured: no`, `server-software: active-directory`, `client-software: sssd`, and a list of `required-package` lines.

**Why:** discovery uses the AD **SRV records** in DNS to find the DCs. It changes nothing, so problems with DNS, the network or packages show up here rather than halfway through a join.

### E2. Join

```
rpm -q sssd-common sssd-ad oddjob oddjob-mkhomedir adcli samba-common-tools
sudo realm join techworks.local
```

`realm join` without `-U` uses the domain account `Administrator`. To use another account with rights to join computers: `sudo realm join -U <domain-admin> techworks.local`, replacing `<domain-admin>` (angle brackets included) with the account name.

**Expected result:** a prompt for the `sudo` password, then `Password for Administrator@TECHWORKS.LOCAL:`, then `Join is successful!` and a note that AD users sign in with fully qualified names (`username@domain`).

| The join | Where |
| --- | --- |
| Creates computer account `TWREDHAT` | AD, default `Computers` container |
| Stores the computer's own credentials | `/etc/krb5.keytab` |
| Configures AD sign-in | `/etc/sssd/sssd.conf` |
| Home folder at first sign-in | `oddjob-mkhomedir` |
| Registers A and PTR records | AD DNS, by `sssd` dynamic update with the computer account |

**Fully qualified names:** AD users sign in as `amy.fin@techworks.local`, not `amy.fin`. This prevents confusion with a local account of the same name. Changeable with `use_fully_qualified_names = False` in `/etc/sssd/sssd.conf`; left at the default here.

> **Roadblock:** the first form tried, `realm join --user=<domain-admin> ...`, did not work. Running `realm` on its own printed its usage: the option is `-U user`, and without it the account defaults to `Administrator`. **Lesson:** when an example fails, read the tool's own help.

### E3. Verify the join

```
realm list
id administrator@techworks.local
id amy.fin@techworks.local
getent ahostsv4 twredhat.techworks.local
```

**Expected result:**

- `configured: kerberos-member`, `login-formats: %U@techworks.local`, `login-policy: allow-realm-logins`.
- `id` shows each user's `uid`, primary group `domain users@techworks.local` and AD groups. For `amy.fin`: `gg_finance` and `dl_finance_rw` (the AGDLP groups from SAM-ADM-001).
- `getent ahostsv4` returns `<rhel-address>`.

**On DC-02:**

```
Active Directory Users and Computers
> techworks.local
> Computers
> TWREDHAT
```

```
DNS
> DC-02
> Forward Lookup Zones
> techworks.local
> F5
> twredhat  Host (A)  <rhel-address>
```

```
DNS
> DC-02
> Reverse Lookup Zones
> <reversed-network>.in-addr.arpa
> F5
> <rhel-address>  Pointer (PTR)  twredhat.techworks.local
```

All three were present in this lab, timestamped at the join.

> **Roadblock:** `id administrator@techworls.local` gave `no such user`. Typo in the domain. `no such user` for an AD account means check the spelling before suspecting the join.

> **Roadblock:** `getent host ...` gave `Unknown database: host`. The database is `hosts` (plural).

> **Roadblock:** `getent hosts twredhat.techworks.local` returned an IPv6 `fe80::` address, not the IPv4 address. The server answered for its **own** name locally (the `myhostname` module in `/etc/nsswitch.conf`) before asking DNS. **Fix:** `getent ahostsv4 <name>` to ask for IPv4 only, or check from another machine with `nslookup twredhat.techworks.local`.

### E4. Sign in as an AD user

```
su - amy.fin@techworks.local
whoami
pwd
id
exit
```

**Expected result:** a password prompt (Amy's **domain** password); `whoami` → `amy.fin@techworks.local`; `pwd` → `/home/amy.fin@techworks.local`; `id` shows her AD groups and an SELinux `context=` line; `exit` returns to the local account. All as expected in this lab.

**Why:** `id` proved the server can *read* AD. This proves an AD user can *sign in*: Kerberos checks the password with a DC, `sssd` authorises the session, and `oddjob-mkhomedir` creates the home folder. One password now works on Windows and Linux.

The `context=unconfined_u:unconfined_r:unconfined_t:...` part of `id` is the user's **SELinux** label. SELinux is RHEL's mandatory access control, enabled (enforcing) by default and an RHCSA topic in its own right.

**Note:** `login-policy: allow-realm-logins` means **every** domain user can sign in. Part F restricts this.

## Part F: Control who can sign in and who gets sudo

**Outcome:** only members of an AD group can sign in to `twredhat`, and the same group gets `sudo`. Membership is managed in AD, not on the server.

**Design (AGDLP, as in SAM-ADM-001):**

| AGDLP | This server |
| --- | --- |
| **A**ccounts | Domain admin accounts (`Administrator`, `<domain-admin>`, `techwork-admin`) |
| **G**lobal group (role) | `gg_linux_admins` |
| **D**omain **L**ocal group (resource) | `dl_twredhat_admins` |
| **P**ermission | Sign-in and `sudo` on `twredhat` |

A future Linux server gets its own `dl_<server>_admins`; nesting `gg_linux_admins` into it grants the same people access without touching any server.

**Two kinds of admin account:**

| Account | Lives in | Purpose |
| --- | --- | --- |
| `<local-admin>` (member of `wheel`) | Local, this server only | **Break-glass**: works when AD is unreachable. Not affected by `realm permit` |
| Domain admin accounts | AD | Day-to-day administration through `gg_linux_admins` |

### F1. Create the groups in AD

**Where:** DC-02, Active Directory Users and Computers, in the OU that holds the other `gg_`/`dl_` groups (`Groups`).

```
> right-click Groups
> New
> Group
> Group name: gg_linux_admins
> Group scope: Global
> Group type: Security
> OK
```

```
> right-click Groups
> New
> Group
> Group name: dl_twredhat_admins
> Group scope: Domain local
> Group type: Security
> OK
```

```
> double-click dl_twredhat_admins
> Members
> Add...
> gg_linux_admins
> Check Names
> OK
> OK
```

```
> double-click gg_linux_admins
> Members
> Add...
> <admin account>
> Check Names
> OK
> OK
```

**Expected result:** `dl_twredhat_admins` contains `gg_linux_admins`; `gg_linux_admins` contains the admin accounts; `amy.fin` is in neither (she is the negative test).

**Automation equivalent** (PowerShell on a DC):

```
New-ADGroup -Name gg_linux_admins -GroupScope Global -GroupCategory Security -Path "<OU distinguished name>"
New-ADGroup -Name dl_twredhat_admins -GroupScope DomainLocal -GroupCategory Security -Path "<OU distinguished name>"
Add-ADGroupMember dl_twredhat_admins -Members gg_linux_admins
Add-ADGroupMember gg_linux_admins -Members <admin account>
```

### F2. Check the server sees the groups

**Where:** the RHEL VM.

```
sudo sss_cache -E
id <domain-admin>@techworks.local
```

**Expected result:** the group list includes `gg_linux_admins@techworks.local` and `dl_twredhat_admins@techworks.local`.

**Why `sss_cache -E`:** `sssd` caches users and groups so sign-ins survive a short DC outage. `-E` expires the cache so the next lookup asks AD. The Linux equivalent of signing out and back in on Windows.

**Linux uses the `SamAccountName`** (the "pre-Windows 2000" logon name), not the display name. To find it on a DC:

```
Get-ADUser -Filter "Name -eq '<display name>'" | Select-Object Name, SamAccountName, UserPrincipalName
```

> **Roadblock:** `id techworks admin@techworks.local` and several variations all gave `no such user`. "techworks admin" is the **display name**; the `SamAccountName` is `techwork-admin` (no `s`). **Fix:** look the name up with `Get-ADUser`, then `id techwork-admin@techworks.local` worked.

> **Roadblock:** `id <techworks admi>n@...` gave `bash: techworks: No such file or directory`. In Bash, `<` means "read input from this file". **Angle brackets in this document only mark the part to replace; never type them.**

### F3. Allow only the group to sign in

```
sudo realm deny --all
sudo realm permit -g dl_twredhat_admins@techworks.local
realm list
```

**Expected result:** `login-policy: allow-permitted-logins` and `permitted-groups: dl_twredhat_admins@techworks.local`.

**Test both ways:**

```
su - amy.fin@techworks.local
su - <domain-admin>@techworks.local
whoami
exit
```

**Expected result:** `amy.fin` gets `su: Permission denied` after a correct password; `<domain-admin>` signs in.

**Why the domain local group:** the permission belongs to the resource group; who is in it is decided in AD.

### F4. Give the group sudo

```
sudo visudo -f /etc/sudoers.d/twredhat-admins
```

```
%dl_twredhat_admins@techworks.local  ALL=(ALL)  ALL
```

```
sudo chmod 0440 /etc/sudoers.d/twredhat-admins
sudo ls -l /etc/sudoers.d/twredhat-admins
sudo visudo -c
```

| Part | Meaning |
| --- | --- |
| `%` | A group |
| `ALL=` | On any host |
| `(ALL)` | As any user, including root |
| `ALL` | Any command; the user's own password is required |

**Expected result:** `-r--r----- 1 root root`; `visudo -c` reports `parsed OK` for each file.

**Test:**

```
su - <domain-admin>@techworks.local
sudo -l
sudo whoami
exit
```

**Expected result:** `sudo -l` shows `(ALL) ALL`; `sudo whoami` prints `root`.

**Why `visudo`:** it checks syntax before saving. A broken sudoers file can lock everyone out of `sudo`, including the break-glass account. **Why `/etc/sudoers.d/`:** a separate file is easy to find, back up, remove and later deploy with Ansible.

> **Roadblock:** `visudo -c` reported `/etc/sudoers.d/twredhat-admins: bad permissions, should be mode 0440`. **Fix:** `sudo chmod 0440` on the file.

> **Roadblock:** `ls -l /etc/sudoers.d/twredhat-admins` gave `Permission denied`. The folder itself is readable only by root. **Fix:** `sudo ls -l ...`. This is intended security, not a fault.

> **Roadblock:** the first `su - <domain-admin>@techworks.local` gave `Authentication failure` (mistyped password). On the next successful sign-in RHEL reported `There was 1 failed login attempt since the last successful login`: a built-in way for users to notice password guessing.

## Part G: Remote access and the login banner

**Outcome:** the server is managed from the desktop (SSH and the Cockpit web console), and every shell shows the server's facts, like BGInfo on Windows.

### G0. Work over SSH or Cockpit instead of the Proxmox console

The Proxmox console does not support copy and paste. A terminal on the desktop does.

**Where:** first on the VM console, then on the desktop.

```
systemctl is-active sshd
sudo firewall-cmd --list-services
```

**Expected result:** `active`; the list includes `ssh` (and `cockpit`). Both are enabled by default on RHEL. If not: `sudo systemctl enable --now sshd` and `sudo firewall-cmd --permanent --add-service=ssh && sudo firewall-cmd --reload`.

From Windows Terminal or PowerShell on the desktop:

```
ssh <local-admin>@<rhel-address>
ssh -l <domain-admin>@techworks.local <rhel-address>
```

**Expected result:** on first connection, `Are you sure you want to continue connecting` → `yes` (the desktop records the server's host key and warns if it ever changes); then the password and a prompt.

**Cockpit** (RHEL's web console): browser → `https://<rhel-address>:9090`, sign in, then **Turn on administrative access** (the `sudo` equivalent). Logs, services, storage, networking, SELinux and a terminal where copy and paste work. If it does not load: `sudo systemctl enable --now cockpit.socket`. The browser warns about the certificate: Cockpit uses a self-signed one by default.

**Pasting into vim:** type `:set paste`, Enter, `i`, paste, `Esc`, `:set nopaste`, Enter, `:wq`. Without paste mode, vim may indent every pasted line.

> **Roadblock:** `ssh <domain-admin>@<rhel-address>` and `ssh Administrator@<rhel-address>` gave `Permission denied` for every password. Without the domain part, SSH asks for a **local** user `<domain-admin>` or `Administrator`, which do not exist. The only local admin is `<local-admin>`. **Fix:** `ssh <domain-admin>@techworks.local@<rhel-address>` (SSH splits at the **last** `@`) or `ssh -l <domain-admin>@techworks.local <rhel-address>`.

### G1. The login banner

```
sudo vim /etc/profile.d/techworks-info.sh
```

```
# Techworks: show server facts at sign-in (managed file, see SAM-ADM-004 Part G)
[ -n "$PS1" ] || return 0
printf '\n  %-9s %s\n' "Host:"    "$(hostname -f)"
printf '  %-9s %s\n'   "OS:"      "$(. /etc/os-release && echo "$PRETTY_NAME")"
printf '  %-9s %s\n'   "Address:" "$(hostname -I | awk '{print $1}')"
printf '  %-9s %s\n'   "Domain:"  "$(realm list --name-only 2>/dev/null | head -1)"
printf '  %-9s %s\n'   "Uptime:"  "$(uptime -p)"
printf '  %-9s %s\n\n' "User:"    "$(whoami)"
```

```
sudo chmod 0644 /etc/profile.d/techworks-info.sh
bash -n /etc/profile.d/techworks-info.sh && echo "syntax OK"
su - <domain-admin>@techworks.local
```

**Expected result:** `syntax OK`, then at sign-in:

```
  Host:     twredhat.techworks.local
  OS:       Red Hat Enterprise Linux 10.3 (Coughlan)
  Address:  <rhel-address>
  Domain:   techworks.local
  Uptime:   up ...
  User:     <domain-admin>@techworks.local
```

| Line | Purpose |
| --- | --- |
| `[ -n "$PS1" ] \|\| return 0` | Interactive shells only, so scripts, `scp` and Ansible are not disturbed |
| `printf '  %-9s %s'` | Aligned labels |
| `hostname -f`, `/etc/os-release`, `hostname -I`, `realm list --name-only`, `uptime -p` | Full name, OS, first address, domain, uptime |

**Why `/etc/profile.d/`:** every `.sh` file there runs for each user's shell (on RHEL, new terminal tabs too). Mode `0644`: readable by all, changeable only by root.

**Next step for Windows:** BGInfo on the DCs and clients through a GPO (planned as its own document).

---

## Part H: Prove it survives a reboot

**Outcome:** everything configured in Parts B to G comes back on its own. Testing before a reboot (Part B3) prevents a broken boot; this proves the boot actually works.

```
sudo systemctl reboot
```

After about a minute, from the desktop:

```
ssh -l <domain-admin>@techworks.local <rhel-address>
findmnt /repo
sudo dnf repolist
ip route | grep default
realm list | grep -E 'configured|permitted'
sudo whoami
```

**Expected result (all passed in this lab):**

| Check | Proves |
| --- | --- |
| SSH as an AD admin works; banner shows `up 0 minutes` | Network, `sssd`, the sign-in rule, `profile.d` |
| `/repo  /dev/sr0  iso9660  ro,...` | The `/etc/fstab` line at boot |
| Both DVD repositories | The repository |
| `proto static` | The static address |
| `kerberos-member`; `permitted-groups: dl_twredhat_admins@techworks.local` | Domain membership and sign-in rule |
| `root` | The sudo rule |

Cockpit as `<domain-admin>@techworks.local` also gained administrative access after the reboot, through the same AD group.

> **Roadblock:** `findmt /repo` gave `command not found`. Typo for `findmnt` (find mount).

---

## 4. Verification

```
hostnamectl
cat /etc/redhat-release
timedatectl | grep synchronized
findmnt /repo
sudo dnf repolist
rpm -q realmd sssd adcli oddjob-mkhomedir samba-common-tools krb5-workstation
```

Plus the Part C3 and E3 checks, and:

```
realm list | grep -E 'login-policy|permitted'
sudo visudo -c
```

`amy.fin` is refused (`su: Permission denied`); a member of `gg_linux_admins` signs in and `sudo whoami` prints `root`.

**Expected result:** the values recorded in Part A; `/repo` mounted from `/dev/sr0` as `iso9660`; both DVD repositories listed; all six packages installed.

## 5. Rollback

| To undo | Do this |
| --- | --- |
| The hostname | `sudo hostnamectl set-hostname <previous-name>` |
| The repository | `sudo rm /etc/yum.repos.d/rhel-dvd.repo` |
| The /repo mount | `sudo umount /repo`, then `sudo cp /etc/fstab.bak-<date> /etc/fstab` and `sudo systemctl daemon-reload` |
| The plugin change | `sudo cp /etc/dnf/plugins/subscription-manager.conf.bak /etc/dnf/plugins/subscription-manager.conf` |
| The packages | `sudo dnf history`, then `sudo dnf history undo <id>` |
| The sign-in restriction | `sudo realm permit --all` |
| The sudo rule | `sudo rm /etc/sudoers.d/twredhat-admins` |
| The AD groups | Delete `dl_twredhat_admins` and `gg_linux_admins` in ADUC (after removing the two rows above) |
| The domain join | `sudo realm leave -U Administrator techworks.local` (removes the computer account), or `sudo realm leave` to leave locally only and delete `TWREDHAT` in AD by hand |
| The login banner | `sudo rm /etc/profile.d/techworks-info.sh` |
| The static address | `sudo nmcli connection modify ens18 ipv4.method auto ipv4.addresses "" ipv4.gateway "" ipv4.dns "" ipv4.dns-search ""`, then `sudo nmcli connection up ens18` (at the console) |

## 6. Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| Prompt still shows `localhost` after setting the hostname | The prompt is set when the shell starts | Open a new terminal tab |
| `Permission denied` running a file in `/etc` | It is a text file, not a program | `cat <file>` |
| `sudo <file>: command not found` | Same: `sudo` runs programs | `cat <file>` |
| "System Not Registered" notice | No subscription, by design | Dismiss it; software comes from the local repository |
| `dnf` prints "not registered" on every command | `subscription-manager` plugin enabled | B5: `enabled=0` in its plugin file |
| `No such file or directory` | Usually a typo in the path | Check spelling (`/var/lib`, not `/var/lig`) |
| `cp: ... Not a directory` | A `/` where a `.` was meant | `/etc/fstab.bak-<date>` |
| `findmnt: unrecognized option` | Option typo | `findmnt --verify` |
| `ls /repo` empty after `mount -a` | ISO not attached, or wrong device | `lsblk`, `blkid /dev/sr0`; Part B2 |
| Server drops to emergency mode after reboot | Bad `/etc/fstab` line | At the emergency prompt: `mount -o remount,rw /`, restore `/etc/fstab.bak-<date>`, reboot |
| `dnf` checksum errors | Damaged ISO | Part B1 hash check |
| Address unchanged after saving in `nmtui` | The profile was saved but not reapplied | Activate a connection > `ens18` > Deactivate, then Activate; or `sudo nmcli connection up ens18` |
| `getent hosts dc-02...` prints nothing | DNS not pointing at the DCs, or wrong name | `cat /etc/resolv.conf`; check the DC's name in AD |
| SSH session froze during the change | The address it was connected to went away | Do network changes at the console |
| `Ping request could not find host` | No A record, or wrong DNS server | `nslookup <name>`; check DNS servers |
| `nslookup <address>`: `Non-existent domain` | No reverse zone, or the PTR is not registered | Part D1; `ipconfig /registerdns` on that machine |
| `'dig' is not recognized` (Windows) | `dig` is a Linux tool | `nslookup` or `Resolve-DnsName` |
| `realm join` rejects `--user=<...>` | Angle brackets typed literally, or option form | `realm` alone shows usage; use `-U <account>` |
| `id <user>@techworks.local`: `no such user` | Typo, or not joined | Check spelling; `realm list` |
| `getent: Unknown database: host` | The database is `hosts` | `getent hosts <name>` |
| `getent hosts <own name>` returns `fe80::...` | The server resolves its own name locally first | `getent ahostsv4 <name>`; or `nslookup` from another machine |
| `id <display name>@techworks.local`: `no such user` | Linux uses the `SamAccountName` | `Get-ADUser -Filter "Name -eq '<name>'"` on a DC |
| New AD group not shown by `id` | `sssd` cache | `sudo sss_cache -E`, then `id` again |
| `bash: <word>: No such file or directory` | Angle brackets typed; `<` is input redirection | Type the value without `< >` |
| `su: Permission denied` with the right password | User not in a permitted group (`realm permit`) | Add the user to `gg_linux_admins` in AD; `sss_cache -E` |
| `visudo -c`: `bad permissions, should be mode 0440` | File mode wrong | `sudo chmod 0440 <file>` |
| `ls` in `/etc/sudoers.d/`: `Permission denied` | Folder readable by root only | `sudo ls -l ...` |
| `<user> is not in the sudoers file` | Group line missing, or the user is not in the group | `sudo visudo -c`; `id <user>@techworks.local` |
| SSH as an AD user: `Permission denied` for every password | Domain part missing; SSH looked for a local user | `ssh -l <user>@techworks.local <address>` |
| Cockpit says "Limited access" | Signed in without admin rights | Turn on administrative access (needs `sudo` rights) |
| Banner missing | Script not in `/etc/profile.d/`, wrong mode, or syntax error | `ls -l /etc/profile.d/techworks-info.sh`; `bash -n` on it |
| `findmt: command not found` | Typo | `findmnt` |
| A test install says "already installed" | Server with GUI includes it | `dnf list --available` and pick something missing |

## 7. Key lessons

> ### Set the final name before anything depends on it.

- **Read files with `cat`; run programs by name.** "Permission denied" on a text file usually means the shell tried to execute it.
- **Check the clock before a domain join.** Kerberos allows only 5 minutes of difference.
- **Test `/etc/fstab` before rebooting** (`findmnt --verify`, `mount -a`), and use `nofail` for removable media.
- **A test only proves something if it can fail.** Installing an already-installed package proves nothing about the repository.
- **Hash a large file twice** to check both the file and the host's memory.
- **Servers get static addresses outside the DHCP pool.** DHCP cannot see a static address inside its pool and may hand it out again.
- **`ping <name>` uses the A record; reverse lookups use the PTR record.**
- **Grant access to AD groups, not to people, and not on the server.** Adding an admin is an AD change, not a server change.
- **Keep a local break-glass admin** for when AD is unreachable.
- **Linux looks AD users up by `SamAccountName`**, not the display name.
- **Edit sudoers only with `visudo`**, in a separate file under `/etc/sudoers.d/`, mode `0440`.
- **Reboot to prove a build.** A server that only works until its next restart is not finished.
- **AD users sign in to Linux with the full name** (`user@techworks.local`), in SSH and Cockpit alike.
- **When an example command fails, read the tool's own help** (`realm` alone printed the correct `-U` option).
- **`proto dhcp` vs `proto static`** in `ip route` shows at a glance where the address came from.

## 8. Command reference

| Task | Command |
| --- | --- |
| Show the hostname | `hostnamectl` or `nmcli general hostname` |
| Set the hostname permanently | `sudo hostnamectl set-hostname <name>.techworks.local` |
| Show the RHEL release | `cat /etc/redhat-release` |
| Show time, time zone and sync status | `timedatectl` |
| CPU count / memory / disks | `nproc` / `free -h` / `lsblk` |
| Network cards and addresses, brief | `ip -br addr` |
| Address method and DNS of the active connection | `nmcli -g ipv4.method,ipv4.dns connection show "<connection>"` |
| Hash a file (run twice) | `sha256sum <file>` |
| Show disks including the DVD | `lsblk` |
| Show a device's label and filesystem type | `sudo blkid /dev/sr0` |
| Check `/etc/fstab` for errors | `sudo findmnt --verify` |
| Mount everything in `/etc/fstab` | `sudo mount -a` |
| List enabled repositories | `sudo dnf repolist` |
| Clear `dnf`'s cache | `sudo dnf clean all` |
| Show a package's details and source | `sudo dnf info <package>` |
| List packages available but not installed | `sudo dnf list --available` |
| Install packages | `sudo dnf install <package> ...` |
| Is a package installed? | `rpm -q <package>` |
| Undo a `dnf` transaction | `sudo dnf history undo <id>` |
| List NetworkManager connections | `nmcli connection show` |
| Menu tool for network settings | `sudo nmtui` |
| Set a static address in one command | `sudo nmcli connection modify ens18 ipv4.method manual ipv4.addresses <addr>/24 ipv4.gateway <gw> ipv4.dns "<dns1> <dns2>" ipv4.dns-search techworks.local` |
| Apply a connection's settings | `sudo nmcli connection up ens18` |
| Default route and its source | `ip route \| grep default` |
| DNS servers in use | `cat /etc/resolv.conf` |
| Resolve a name the way the system does | `getent hosts <name>` |
| Forward / reverse lookup (Windows or Linux) | `nslookup <name>` / `nslookup <address>` |
| Re-register a Windows machine's DNS records | `ipconfig /registerdns` |
| Create a reverse zone (PowerShell) | `Add-DnsServerPrimaryZone -NetworkId "<net>/24" -ReplicationScope Domain -DynamicUpdate Secure` |
| Find the domain (read-only) | `realm discover techworks.local` |
| Join the domain | `sudo realm join [-U <account>] techworks.local` |
| Show join status and login policy | `realm list` |
| Show an AD user's IDs and groups | `id <user>@techworks.local` |
| Resolve a name, IPv4 only | `getent ahostsv4 <name>` |
| Expire the `sssd` cache | `sudo sss_cache -E` |
| Find an AD user's logon names (PowerShell) | `Get-ADUser -Filter "Name -eq '<name>'" \| Select-Object Name, SamAccountName, UserPrincipalName` |
| Allow only one group to sign in | `sudo realm deny --all` then `sudo realm permit -g <group>@techworks.local` |
| Allow all domain users again | `sudo realm permit --all` |
| Edit a sudoers drop-in safely | `sudo visudo -f /etc/sudoers.d/<file>` |
| Check all sudoers files | `sudo visudo -c` |
| What may I run with sudo? | `sudo -l` |
| Sign in as an AD user | `su - <user>@techworks.local` |
| SSH as an AD user | `ssh -l <user>@techworks.local <address>` |
| Is a service running? | `systemctl is-active <service>` |
| Firewall: allowed services | `sudo firewall-cmd --list-services` |
| Allow a service permanently | `sudo firewall-cmd --permanent --add-service=<name> && sudo firewall-cmd --reload` |
| Start Cockpit's web console | `sudo systemctl enable --now cockpit.socket` (then `https://<address>:9090`) |
| Check a shell script's syntax | `bash -n <file>` |
| Reboot | `sudo systemctl reboot` |
| Leave the domain | `sudo realm leave -U <account> techworks.local` |
| Add a repository (exam shortcut) | `sudo dnf config-manager --add-repo file:///repo/BaseOS` |

## 9. Design notes

- **No subscription.** Software comes from the BaseOS and AppStream repositories on the installation ISO, the method used in the RHCSA exam. Trade-off: no security updates. Red Hat's free Developer subscription can add them later.
- **`pve02` capacity.** The host runs at about 84% RAM with this VM; plan memory before adding the email and ticketing VMs.
- **`pve02` Proxmox updates.** The Proxmox Enterprise repository is enabled without a subscription (same issue as RB-Proxmox-018), so `pve02` is not receiving Proxmox updates.
- **Placement.** VM 500 on `pve02`. `pve03` was ruled out after a memory test found a failing RAM module (it was corrupting the ISO when read).

## 10. Open items

- [x] Part A: hostname and facts (2026-11-24).
- [x] Part B: local repository from the ISO (2026-11-24).
- [x] Part C: static address; DNS pointing at the DCs (2026-11-24).
- [x] Part D: reverse lookup zone; DC PTR records (2026-11-24). `twredhat` A/PTR to be created by the join (Part E).
- [x] Part E1–E3: domain discovered and joined; users, groups and DNS records verified (2026-11-24).
- [x] Part E4: AD user sign-in tested (2026-11-24).
- [x] Part F: sign-in limited to `dl_twredhat_admins`; `sudo` for the same group (2026-11-25).
- [x] Part G: SSH and Cockpit access; login banner (2026-11-25).
- [x] Part H: reboot test passed (2026-11-25).
- [ ] BGInfo on the Windows DCs and clients through a GPO (own document).
- [ ] Fix `pve02`'s Proxmox repositories (RB-Proxmox-018 method).
- [ ] Find where the stray DNS suffix `ourhom3.arpa` comes from (DC-02 network settings or pfSense System > General Setup > Domain) and replace it with `techworks.local`.
- [ ] **DNS cleanup (found during E3):** stale Name Server (NS) record for an old DC name (`win-...`) in the `techworks.local` zone; two A records on the same address (`TW-CL02` and an old `Techworks00`). Remove stale records, then turn on aging and scavenging so old records expire automatically.
- [ ] **Security finding:** the group `Workstation-Admins` is a member of **Domain Admins**, so every workstation admin is a full domain admin. Remove it from Domain Admins; grant local admin on workstations through a GPO (Restricted Groups or Local Users and Groups) instead.
- [ ] Optional: give `techwork-admin` a clearer display name or matching logon name, to avoid confusion.
- [ ] Move `TWREDHAT` from the default `Computers` container to a servers OU (OUs can carry GPOs; containers cannot).
- [ ] Run `ipconfig /registerdns` on the Windows clients so they get PTR records.
- [ ] Future topic: how DCs reach the internet safely (DNS forwarders, outbound-only rules, Tier 0).
- [ ] Optional: register with the free Red Hat Developer subscription for security updates.

## Document history

| Date | Change | By |
| --- | --- | --- |
| 2026-11-24 | First issue: Part A | Techworks |
| 2026-11-24 | Added Part B (local ISO repository) | Techworks |
| 2026-11-24 | Added Part C (static address, DNS on the DCs) | Techworks |
| 2026-11-24 | Added Part D (reverse lookup zone, PTR records) | Techworks |
| 2026-11-24 | Added Part E (domain join, verification, AD sign-in) | Techworks |
| 2026-11-25 | Added Part F (sign-in and sudo by AD group) | Techworks |
| 2026-11-25 | Added Parts G (SSH, Cockpit, banner) and H (reboot test). Complete | Techworks |
