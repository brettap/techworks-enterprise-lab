# SAM-ADM-001: Simulating Windows User Activity

| Field | Value |
| --- | --- |
| Document ID | SAM-ADM-001 |
| Title | Simulating Windows user activity in an Active Directory lab |
| Owner / Author | Techworks |
| Status | Built and verified (Parts A to F). Later phases listed under Open Items |
| Last updated | 2026-11-16 |
| Environment | `techworks.local` lab domain, VMware Workstation clients, Proxmox-hosted domain controllers |

## How to read this document

This document is layered so that each reader can stop at the depth they need.

| If you are | Read |
| --- | --- |
| Level 1 / Level 2 technician | Purpose, Prerequisites, the **Steps** and **Expected result** in each Part, Verification, Troubleshooting |
| Systems administrator / engineer | All of the above, plus the **Why** notes, **Automation equivalent**, and Rollback |
| Architect / reviewer | Purpose, Architecture, Design Notes, Open Items |

Conventions used throughout:

- Click paths are written one click per line, each starting with `>`.
- Every procedure gives the manual method first. The scripted method follows under **Automation equivalent**.
- Every procedure states an **Expected result**. If you do not see it, stop and go to Troubleshooting.
- **Roadblock** notes record problems that were actually hit while building this, with the cause and fix.

---

## 1. Purpose and outcome

Three Windows 11 clients behave like staff at work every weekday morning, with no one at the keyboard, so that the domain produces realistic logon, file access and "access denied" events to practice help desk and systems administration work on.

**You know it works when**, on a weekday between 8:05 AM and noon:

- Three client VMs are running.
- Each is signed in as its own simulated user.
- Notepad opens and types on each client at intervals.
- Each user's `Documents\simulation.log` gains new lines.
- The file server's Security log shows events 4663 (file access) and 4656 (access refused) under those users' names.

## 2. Scope and environment

| Item | Value |
| --- | --- |
| Domain | `techworks.local` (NetBIOS `TECHWORKS`) |
| Domain controllers | `Techworks-DC01`, `DC-02` (Windows Server 2025) |
| File server | `Techworks-DC01` (lab shortcut; see Design Notes) |
| Clients | `TW-CL00`, `TW-CL01`, `TW-CL02` (Windows 11, VMware Workstation) |
| Client OU | `OU=Workstations,DC=techworks,DC=local` |
| VMware host | A Windows PC that is **not** domain joined |
| Active hours | 8:00 AM to 12:00 PM Eastern, Monday to Friday |

Simulated users:

| Account | Name | Department | Client | Drive |
| --- | --- | --- | --- | --- |
| `amy.fin` | Amy Carter | Finance | TW-CL00 | `F:` Finance |
| `ben.gis` | Ben Ortiz | GIS | TW-CL01 | `M:` GIS |
| `cara.ops` | Cara Singh | Operations | TW-CL02 | `O:` Operations, `M:` GIS (read-only) |

Out of scope: email, the accounting and GIS applications, ticketing, and fault injection. See Open Items.

## 3. Architecture

The daily cycle, and where each piece lives:

| Order | What happens | Mechanism | Stored at |
| --- | --- | --- | --- |
| 1 | Clients power on at 8:00 AM | Scheduled task `Lab\Clients-Start` runs `vmrun` | VMware host: `C:\LabScripts` |
| 2 | Each client signs in as its user | Sysinternals Autologon | Each client's registry (password stored as an LSA secret) |
| 3 | Policy applies: no lock, no sleep, drive mapped | Group Policy | GPOs `Sim - Client Settings`, `Sim - User Activity` |
| 4 | One minute after logon, the activity script starts | Scheduled task `Sim Typing` | Each client's Task Scheduler |
| 5 | Script types in Notepad, then does one file action, then idles 5 to 20 minutes, repeating until 11:50 AM | PowerShell | `\\TECHWORKS\NETLOGON\Sim\Invoke-SimTyping.ps1` |
| 6 | File access and refusals are recorded | Audit policy plus audit entries on folders | GPO `Sim - Audit Policy`; Security log on `Techworks-DC01` |
| 7 | Clients shut down at 12:00 PM | Scheduled task `Lab\Clients-Stop` | VMware host: `C:\LabScripts` |

A separate, older task named `TechWorks-UserSimulation` runs as `SYSTEM` on TW-CL00 and performs machine health checks (DNS lookup, LDAP port test). It was left in place. It does not produce user-attributed events.

## 4. Prerequisites

- A domain admin account, used from a **domain-joined** machine (the DC).
- PowerShell remoting enabled on the clients (here via `GPO-Workstation-Remote-Management`).
- VMware Tools installed in every client VM. It is required for clean shutdown and copy/paste.
- The VM encryption password. Windows 11 guests have a virtual TPM, which forces VM encryption.
- The VMware host stays powered on, awake, on AC power, with the task owner signed in (a locked screen is fine).

---

## Part A: Active Directory groundwork

**Outcome:** a `Sim` OU with three users, and three GPOs controlling auditing, client behavior and user settings.

### A1. Create the OU structure and users

**Steps (manual):**

```
Server Manager
> Tools
> Active Directory Users and Computers
> right-click techworks.local
> New
> Organizational Unit
> Name: Sim
> OK
```

Repeat inside `Sim` to create the OUs `Users`, `Computers` and `Groups`. Then for each user:

```
Active Directory Users and Computers
> Sim
> right-click Users
> New
> User
> fill in First name, Last name, User logon name (for example amy.fin)
> Next
> enter the password
> untick "User must change password at next logon"
> Next
> Finish
```

**Expected result:** `Sim > Users` lists Amy Carter, Ben Ortiz and Cara Singh.

**Automation equivalent** (run on a DC in an elevated PowerShell window):

```powershell
$base = (Get-ADDomain).DistinguishedName
$dns  = (Get-ADDomain).DNSRoot

New-ADOrganizationalUnit -Name 'Sim' -Path $base
'Users','Computers','Groups' | ForEach-Object {
    New-ADOrganizationalUnit -Name $_ -Path "OU=Sim,$base"
}

$pw = Read-Host 'Password for sim users' -AsSecureString
$staff = @(
    @{Sam='amy.fin';  Name='Amy Carter';  Dept='Finance'},
    @{Sam='ben.gis';  Name='Ben Ortiz';   Dept='GIS'},
    @{Sam='cara.ops'; Name='Cara Singh';  Dept='Operations'}
)
foreach ($s in $staff) {
    New-ADUser -Name $s.Name -SamAccountName $s.Sam `
        -UserPrincipalName "$($s.Sam)@$dns" -Department $s.Dept `
        -Path "OU=Users,OU=Sim,$base" -AccountPassword $pw -Enabled $true
}
```

> **Roadblock:** the first version of this script hard-coded a placeholder domain (`DC=lab,DC=local`). Every command failed with "The server is unwilling to process the request". **Fix:** read the real values with `Get-ADDomain`, as above. Never paste a distinguished name from an example without checking it.

### A2. GPO: Sim - Audit Policy

**Linked to:** `Sim`, `Workstations`, `Domain Controllers`.

**Steps:**

```
Group Policy Management
> right-click Sim
> Create a GPO in this domain, and Link it here
> Name: Sim - Audit Policy
> right-click Sim - Audit Policy
> Edit
> Computer Configuration
> Policies
> Windows Settings
> Security Settings
> Advanced Audit Policy Configuration
> Audit Policies
```

Configure these subcategories (double-click each, tick "Configure the following audit events"):

| Category | Subcategory | Setting |
| --- | --- | --- |
| Account Logon | Kerberos Authentication Service, Credential Validation | Success, Failure |
| Account Management | User Account Management, Security Group Management | Success, Failure |
| Logon/Logoff | Logon, Logoff, Account Lockout | Success, Failure |
| Object Access | File System, File Share, Detailed File Share | Success, Failure |
| Object Access | **Handle Manipulation** | **Failure only** |
| Detailed Tracking | Process Creation | Success |

Then:

```
Computer Configuration
> Policies
> Administrative Templates
> System
> Audit Process Creation
> double-click Include command line in process creation events
> Enabled
> OK
```

Link the GPO to the other two OUs:

```
Group Policy Management
> right-click Workstations
> Link an Existing GPO
> select Sim - Audit Policy
> OK
```

Repeat for `Domain Controllers`.

**Expected result:** on a DC, `auditpol /get /category:"Account Logon","Account Management"` shows the four subcategories as `Success and Failure`.

**Why:** the DCs record the most useful account events (Kerberos logons, lockouts, group changes), so the policy must reach them, not only the clients. Once a GPO uses Advanced Audit Policy, every subcategory not explicitly enabled shows `No Auditing`.

> **Roadblock:** refused attempts to open a folder were not being logged at all. **Cause:** event 4656 is only written when Handle Manipulation auditing is on. **Fix:** enable Handle Manipulation for Failure only. Success would log every file handle on the server and bury the useful events.

### A3. GPO: Sim - Client Settings

**Linked to:** `Sim`, `Workstations`.

**Steps:**

```
Group Policy Management
> right-click Workstations
> Create a GPO in this domain, and Link it here
> Name: Sim - Client Settings
> right-click Sim - Client Settings
> Edit
```

| What it does | Path under Computer Configuration > Policies | Setting | Value |
| --- | --- | --- | --- |
| Stops auto-lock | Windows Settings > Security Settings > Local Policies > Security Options | Interactive logon: Machine inactivity limit | Define, `0` seconds |
| Stops sleep | Administrative Templates > System > Power Management > Sleep Settings | Specify the system sleep timeout (plugged in) | Enabled, `0` |
| No password on wake | Administrative Templates > System > Power Management > Sleep Settings | Require a password when a computer wakes (plugged in) | Disabled |
| No password on wake | Same folder | Require a password when a computer wakes (on battery) | Disabled |
| Keeps display on | Administrative Templates > System > Power Management > Video and Display Settings | Turn off the display (plugged in) | Enabled, `0` |
| Real logon, not cached | Administrative Templates > System > Logon | Always wait for the network at computer startup and logon | Enabled |

**Expected result:** on a client, in an elevated prompt, `gpresult /r /scope computer` lists `Sim - Client Settings` under Applied Group Policy Objects.

**Why:** scripted keystrokes only work in an unlocked, awake session. "Always wait for the network" is explained in Part E.

> **Roadblock:** the GPO did not appear in `gpresult`. **Cause 1:** `gpresult /r` only shows the computer half from an elevated prompt. **Cause 2:** the GPO was linked to `Sim\Computers`, which held no computers; the clients were in `Workstations`. **Fix:** link the GPO to the OU that actually contains the objects. Linking was chosen over moving the computers, because `Workstations` carries other GPOs the clients need.

> **Roadblock:** the wake-password setting was first configured as the "(on battery)" variant set to Enabled. **How it was found:** `gpresult /h C:\gp.html /scope computer /f`, then reading the Setting and Winning GPO columns. Policies (Administrative Templates) override Preferences (the existing `Workstations - Power Management` power plan).

### A4. GPO: Sim - User Activity

**Linked to:** `Sim\Users`.

```
Group Policy Management
> Sim
> right-click Users
> Create a GPO in this domain, and Link it here
> Name: Sim - User Activity
> right-click Sim - User Activity
> Edit
> User Configuration
> Policies
> Administrative Templates
> Control Panel
> Personalization
> double-click Enable screen saver
> Disabled
> OK
> double-click Password protect the screen saver
> Disabled
> OK
```

**Expected result:** signed in as a Sim user, in a normal prompt, `gpresult /r /scope user` lists `Sim - User Activity`.

**Why:** screen saver settings are user settings. They follow the account, so they belong in a GPO linked to the users' OU.

> **Key concept:** every GPO has a computer half and a user half. The computer half applies at startup, based on the computer's OU. The user half applies at logon, based on the user's OU. `gpresult /scope computer` and `/scope user` are two views of the same machine and will list different GPOs.

---

## Part B: Power schedule on the VMware host

**Outcome:** clients start at 8:00 AM and shut down at 12:00 PM, Monday to Friday.

### B1. Store the VM encryption password

On the VMware host, signed in as the account that will own the tasks, in PowerShell:

```powershell
New-Item C:\LabScripts -ItemType Directory -Force
Read-Host 'VM encryption password' -AsSecureString | ConvertFrom-SecureString | Set-Content C:\LabScripts\vmpw.txt
```

**Expected result:** `C:\LabScripts\vmpw.txt` contains one long line of letters and numbers. The password is not readable in it.

**Why:** `vmrun` prompts for the password of an encrypted VM, which blocks automation. This stores it encrypted with Windows DPAPI. Only the same Windows account on the same PC can decrypt it.

### B2. Create the scripts

`C:\LabScripts\Start-Clients.ps1`:

```powershell
$vmrun = 'C:\Program Files\VMware\VMware Workstation\vmrun.exe'
$sec   = Get-Content C:\LabScripts\vmpw.txt | ConvertTo-SecureString
$pw    = [System.Net.NetworkCredential]::new('', $sec).Password
$vms = 'C:\ISOs\Windows 11 x64.vmx',
       'C:\clones\Techworks01.vmx',
       'G:\copytest\Clones01\Techworks02.vmx'
foreach ($vm in $vms) {
    & $vmrun -T ws -vp $pw start $vm nogui
    Start-Sleep -Seconds (Get-Random -Minimum 30 -Maximum 120)   # staggered arrivals
}
```

`C:\LabScripts\Stop-Clients.ps1`:

```powershell
$vmrun = 'C:\Program Files\VMware\VMware Workstation\vmrun.exe'
$sec   = Get-Content C:\LabScripts\vmpw.txt | ConvertTo-SecureString
$pw    = [System.Net.NetworkCredential]::new('', $sec).Password
$vms = 'C:\ISOs\Windows 11 x64.vmx',
       'C:\clones\Techworks01.vmx',
       'G:\copytest\Clones01\Techworks02.vmx'
foreach ($vm in $vms) { & $vmrun -T ws -vp $pw stop $vm soft }
```

**Why:** `nogui` starts the VM in the background. `soft` asks Windows to shut down cleanly through VMware Tools. The random pause staggers logon times so they look natural in the logs and avoids starting three VMs at the same instant.

### B3. Register the scheduled tasks

```
Start
> type: cmd
> right-click Command Prompt
> Run as administrator
```

```
schtasks /Create /TN "Lab\Clients-Start" /SC WEEKLY /D MON,TUE,WED,THU,FRI /ST 08:00 /TR "powershell -ExecutionPolicy Bypass -File C:\LabScripts\Start-Clients.ps1"
schtasks /Create /TN "Lab\Clients-Stop"  /SC WEEKLY /D MON,TUE,WED,THU,FRI /ST 12:00 /TR "powershell -ExecutionPolicy Bypass -File C:\LabScripts\Stop-Clients.ps1"
```

**Expected result:** each command replies `SUCCESS`. `schtasks /Query /TN "Lab\Clients-Start"` shows Status `Ready` and the next weekday at 8:00 AM.

### B4. Verify

```
schtasks /Run /TN "Lab\Clients-Start"
```

```powershell
& "C:\Program Files\VMware\VMware Workstation\vmrun.exe" list
schtasks /Query /TN "Lab\Clients-Start" /V /FO LIST | findstr /C:"Status" /C:"Last Result" /C:"Run As User"
```

**Expected result:** `Total running VMs: 3`, Status `Ready` after the run, `Last Result: 0`.

### Roadblocks in Part B

| Symptom | Cause | Fix |
| --- | --- | --- |
| No `Lab` folder in Task Scheduler; `schtasks /Query` says "cannot find the file specified" | The create command was run in a non-elevated prompt | Run Command Prompt as administrator. An elevated prompt opens at `C:\Windows\System32>` |
| `vmrun.exe` "is not recognized" | The documented path (`Program Files (x86)`) was wrong for this install; a 0-byte file sat there | `where /R "C:\Program Files" vmrun.exe` to find the real one |
| Stop script did nothing, with no error | It used `$vms` and `$vmrun` without defining them. Each script runs in its own session | Define the variables in every script |
| Path had a stray space (`C:\Program Files \VMware`) | Hand edit | `type <file>` after every edit to confirm |
| "Encrypted virtual machine password:" prompt | Windows 11 VMs are encrypted because of the virtual TPM | `-vp` with the DPAPI-stored password (B1) |
| `"C:\...\vmrun.exe" list` fails in PowerShell with "Unexpected token" | In PowerShell a quoted path is treated as text | Prefix with the call operator: `& "C:\...\vmrun.exe" list` |
| `Last Result: -2147020576` (`0x800710E0`) | The task was started a second time while still running; the second request was refused | Check `Status` first. `Running` means wait |
| Full start took up to 30 minutes | Original pause was 60 to 600 seconds per VM | Reduced to 30 to 120 seconds |

---

## Part C: Automatic sign-in

**Outcome:** each client signs in as its assigned user at boot.

### C1. Manual method (one client)

Signed in to the client as an administrator:

```
Browser
> https://learn.microsoft.com/sysinternals/downloads/autologon
> Download Autologon
> extract AutoLogon.zip
> right-click Autologon64.exe
> Run as administrator
> Username: amy.fin
> Domain: TECHWORKS
> Password: the user's password
> Enable
```

**Expected result:** "Autologon successfully configured." After a restart the client goes straight to the user's desktop.

> **Roadblock:** TW-CL00 signed in as Administrator after this. **Cause:** the tool pre-fills Username with whoever is running it, and the field was not changed. This also left a domain admin password stored on a client. **Fix:** re-run for the correct user, which overwrites the stored credentials. Always read the three fields back before clicking Enable.

### C2. Automation equivalent (all clients, from the DC)

Download once on the DC:

```powershell
New-Item C:\Tools -ItemType Directory -Force
Invoke-WebRequest https://download.sysinternals.com/files/AutoLogon.zip -OutFile C:\Tools\AutoLogon.zip
Expand-Archive C:\Tools\AutoLogon.zip -DestinationPath C:\Tools\AutoLogon -Force
```

Per client (change the first two lines):

```powershell
$pc   = 'TW-CL01'
$user = 'ben.gis'
$sec  = Read-Host "Password for $user" -AsSecureString
$pw   = [System.Net.NetworkCredential]::new('', $sec).Password

$s = New-PSSession -ComputerName $pc
Copy-Item C:\Tools\AutoLogon\Autologon64.exe -Destination C:\Windows\Temp\ -ToSession $s
Invoke-Command -Session $s -ScriptBlock {
    Start-Process C:\Windows\Temp\Autologon64.exe -ArgumentList $using:user, 'TECHWORKS', $using:pw, '/accepteula' -Wait
    Remove-Item C:\Windows\Temp\Autologon64.exe
    Get-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon' |
        Select-Object AutoAdminLogon, DefaultUserName, DefaultDomainName
}
Remove-PSSession $s

Restart-Computer -ComputerName $pc -Force -Wait -For PowerShell
Start-Sleep -Seconds 60
Invoke-Command -ComputerName $pc -ScriptBlock { quser }
```

**Expected result:** `AutoAdminLogon 1`, the right `DefaultUserName`, and after the restart a `quser` line showing that user at `console`, `Active`.

**Why remoting and not a GPO:** each client needs a different account, and a GPO can only set auto logon by writing the password in plain text to the registry and SYSVOL.

> **Roadblock:** every remote command failed with `0x8009030e ... A specified logon session does not exist`. **Cause:** the commands were run from the VMware host, which is not domain joined, as a local account. It had no domain identity to present. **Fix:** run from the DC. **Habit:** run `hostname` and `whoami` before any command that changes other machines.

---

## Part D: The activity script and its task

**Outcome:** one minute after a Sim user signs in, a script starts typing in Notepad and working with files.

### D1. Place the script on NETLOGON

On the DC, with Notepad run **as administrator**:

```
Start
> type: Notepad
> right-click Notepad
> Run as administrator
> File
> Save As
> File name: \\techworks.local\NETLOGON\Sim\Invoke-SimTyping.ps1
> Save as type: All files
> Save
```

The full script is in Appendix A.

**Why NETLOGON:** every DC publishes it and every domain user can read it, so one copy serves all clients. Edit it once and every client picks up the change.

> **Roadblock:** a Notepad edit to this file silently failed to save. **Cause:** Notepad was not elevated, so the admin group was switched off by UAC and the write was refused. **Proof:** an Audit Failure event 5145 on the DC naming the account, the `NETLOGON` share and `Sim\Invoke-SimTyping.ps1`. **Fix:** run the editor as administrator.

### D2. Test by hand

On a client, signed in as a Sim user:

```
powershell -File \\TECHWORKS\NETLOGON\Sim\Invoke-SimTyping.ps1 -Minutes 2
```

**Expected result:** Notepad opens, words are typed one letter at a time, Notepad closes, and a line is added to `Documents\simulation.log`. Close the prompt afterwards to end the test.

**Why `-Minutes`:** without it the script stops at 11:50 AM, so an afternoon test would exit immediately.

> **Roadblock:** "File ... is not digitally signed. You cannot run this script." **Cause:** `GPO-Workstation-Powershell-Policy` sets the execution policy to `RemoteSigned` at machine level, which overrides `-ExecutionPolicy Bypass`. A path with dots in the server name (`\\techworks.local\...`) is treated as the Internet zone, so the script counted as remote. **Fix:** use the short domain name, `\\TECHWORKS\NETLOGON\...`, which Windows treats as the local intranet. **Check with:** `Get-ExecutionPolicy -List` and read the `MachinePolicy` row.

### D3. Create the scheduled task on each client

**Manual method** (on the client, Task Scheduler run as administrator):

```
Task Scheduler
> right-click Task Scheduler Library
> Create Task
> General tab
> Name: Sim Typing
> Change User or Group
> type: Users
> Check Names
> OK
> Triggers tab
> New
> Begin the task: At log on
> Any user
> Delay task for: 1 minute
> OK
> Actions tab
> New
> Program/script: powershell.exe
> Add arguments: -WindowStyle Hidden -File \\TECHWORKS\NETLOGON\Sim\Invoke-SimTyping.ps1
> OK
> Conditions tab
> untick Start the task only if the computer is on AC power
> OK
```

**Automation equivalent** (from the DC):

```powershell
Invoke-Command -ComputerName TW-CL00, TW-CL01, TW-CL02 -ScriptBlock {
    $action    = New-ScheduledTaskAction -Execute 'powershell.exe' -Argument '-WindowStyle Hidden -File \\TECHWORKS\NETLOGON\Sim\Invoke-SimTyping.ps1'
    $trigger   = New-ScheduledTaskTrigger -AtLogOn
    $trigger.Delay = 'PT1M'
    $principal = New-ScheduledTaskPrincipal -GroupId 'BUILTIN\Users'
    $settings  = New-ScheduledTaskSettingsSet -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries
    Register-ScheduledTask -TaskName 'Sim Typing' -Action $action -Trigger $trigger -Principal $principal -Settings $settings -Force | Out-Null
    "$env:COMPUTERNAME : " + (Get-ScheduledTask -TaskName 'Sim Typing').State
}
```

**Expected result:** each client reports `Ready`. After a restart, typing begins about a minute after the desktop appears.

**Why `BUILTIN\Users`:** naming a group as the task's account means "run as whoever signs in". The script's second line then exits for anyone who is not a Sim user, so administrators are not affected.

> **Roadblock: Group Policy Preferences could not deliver this task.** Two attempts failed, and `gpupdate` reported success both times.
>
> | Attempt | Error (event 4098, source "Group Policy Scheduled Tasks", client Application log) | Cause |
> | --- | --- | --- |
> | User Configuration preference, run as `%LogonDomain%\%LogonUser%` | `0x80070005 Access is denied` | A standard user cannot register a logon-triggered task |
> | Computer Configuration preference, run as `BUILTIN\Users` | `0x80041318 The task XML contains a value which is incorrectly formatted` | Preferences mishandle a group as the task account |
>
> **Fix:** register the task directly on each client, as above. **Lesson:** a failed preference item is written to the client's Application log as event 4098. `gpupdate` success does not mean every item applied.
>
> Also note: in the Preferences menu, choose **Scheduled Task (At least Windows 7)**. The plain "Scheduled Task" entry is the Windows XP format and has different tabs.

---

## Part E: File shares, permissions, auditing and drive maps

**Outcome:** three department shares where each user reaches only their own department, with access and refusals logged.

### E1. Security groups (AGDLP)

Accounts go into Global groups, Global groups go into Domain Local groups, and Domain Local groups hold the Permissions.

| Group | Scope | Members | Grants |
| --- | --- | --- | --- |
| `GG_Finance`, `GG_GIS`, `GG_Operations` | Global | The department's users | Nothing directly |
| `DL_<Dept>_RW` | Domain local | `GG_<Dept>` | Modify on the department folder |
| `DL_<Dept>_RO` | Domain local | `DL_GIS_RO` contains `GG_Operations` | Read on the department folder |
| `DL_Shares_Admin` | Domain local | Lab admin account | Full control on all three folders |

**Steps (manual, per group):**

```
Active Directory Users and Computers
> Sim
> right-click Groups
> New
> Group
> Group name: GG_Finance
> Group scope: Global
> Group type: Security
> OK
> double-click the group
> Members tab
> Add
> type the member name
> Check Names
> OK
```

**Automation equivalent:**

```powershell
$g = 'OU=Groups,OU=Sim,DC=techworks,DC=local'
foreach ($d in 'Finance','GIS','Operations') {
    New-ADGroup -Name "GG_$d"      -GroupScope Global      -GroupCategory Security -Path $g
    New-ADGroup -Name "DL_${d}_RW" -GroupScope DomainLocal -GroupCategory Security -Path $g
    New-ADGroup -Name "DL_${d}_RO" -GroupScope DomainLocal -GroupCategory Security -Path $g
    Add-ADGroupMember "DL_${d}_RW" -Members "GG_$d"
}
Add-ADGroupMember GG_Finance    -Members amy.fin
Add-ADGroupMember GG_GIS        -Members ben.gis
Add-ADGroupMember GG_Operations -Members cara.ops
Add-ADGroupMember DL_GIS_RO     -Members GG_Operations
```

### E2. Folders, permissions and shares

**Steps (manual, per folder, on the file server):**

```
File Explorer
> C:\Shares
> New folder: Finance
> right-click Finance
> Properties
> Security tab
> Advanced
> Disable inheritance
> Remove all inherited permissions from this object
> Add
> Select a principal
> type the group name
> OK
> tick the permission level
> OK
```

Permissions per folder: `SYSTEM` Full control, `Administrators` Full control, `DL_<Dept>_RW` Modify, `DL_<Dept>_RO` Read & execute, `DL_Shares_Admin` Full control.

```
Properties
> Sharing tab
> Advanced Sharing
> tick Share this folder
> Permissions
> remove Everyone
> Add: Authenticated Users
> Full Control
> OK
```

**Automation equivalent:**

```powershell
foreach ($d in 'Finance','GIS','Operations') {
    $p = "C:\Shares\$d"
    New-Item $p -ItemType Directory -Force | Out-Null
    icacls $p /inheritance:r /grant 'SYSTEM:(OI)(CI)F' 'BUILTIN\Administrators:(OI)(CI)F' "TECHWORKS\DL_${d}_RW:(OI)(CI)M" "TECHWORKS\DL_${d}_RO:(OI)(CI)RX" "TECHWORKS\DL_Shares_Admin:(OI)(CI)F" | Out-Null
    New-SmbShare -Name $d -Path $p -FullAccess 'Authenticated Users' -FolderEnumerationMode AccessBased | Out-Null
}
```

**Expected result:** `icacls C:\Shares\Finance` lists only the five entries above. There is no line for `Users`, `Everyone` or an individual person.

**Why:** the share permission is open and the folder permission restricts. Windows applies whichever is stricter. Access comes from groups only, so removing someone from a group reliably removes their access.

> **Roadblock:** a domain admin was refused access to the folders in File Explorer on the server. **Cause:** User Account Control. In a normal window, `BUILTIN\Administrators` and `Domain Admins` are in the token as "Group used for deny only" and cannot grant access. **Proof:** `whoami /groups | findstr /i "admin"` in a normal window versus an elevated one. **Fix:** a custom group (`DL_Shares_Admin`) with Full control. UAC only switches off the built-in admin groups.
>
> **Do not** click **Continue** on "You don't currently have permission to access this folder". It permanently adds your personal account to the folder's permissions.
>
> When adding a group in the Security dialog, Windows grants only Read by default. Tick **Full control** yourself.

Folder permissions can also be edited from an elevated console:

```
Server Manager
> Tools
> Computer Management
> System Tools
> Shared Folders
> Shares
> double-click the share
> Security tab
> Edit
```

The same console's **Sessions** and **Open Files** show who is connected and which files are open, which answers "the file is locked by another user" tickets.

### E3. Audit entries on the folders

```
File Explorer
> right-click C:\Shares\Finance
> Properties
> Security tab
> Advanced
> Auditing tab
> Add
> Select a principal: Everyone
> Type: All
> Applies to: This folder, subfolders and files
> tick Modify
> OK
```

**Automation equivalent:**

```powershell
'Finance','GIS','Operations' | ForEach-Object {
    $p    = "C:\Shares\$_"
    $acl  = Get-Acl $p -Audit
    $rule = New-Object System.Security.AccessControl.FileSystemAuditRule('Everyone', 'Modify', 'ContainerInherit,ObjectInherit', 'None', 'Success,Failure')
    $acl.SetAuditRule($rule)
    Set-Acl $p $acl
}
```

**Why:** the audit policy (A2) allows the server to record file access. The audit entry on each folder says which actions, by whom, to record. Both are required. "Everyone" here means "watch everyone"; it grants no access.

> **Note:** a `foreach (...) { }` statement cannot be piped to another command; PowerShell reports "An empty pipe element is not allowed". Use `ForEach-Object` when the output needs to go down the pipeline.

### E4. Drive maps

In `Sim - User Activity`:

```
Group Policy Management
> Sim
> Users
> right-click Sim - User Activity
> Edit
> User Configuration
> Preferences
> Windows Settings
> right-click Drive Maps
> New
> Mapped Drive
> General tab
> Action: Update
> Location: \\Techworks-DC01\Finance
> tick Reconnect
> Label as: Finance
> Drive Letter: Use: F
> Common tab
> tick Run in logged-on user's security context
> tick Item-level targeting
> Targeting
> New Item
> Security Group
> ... button
> type: GG_Finance
> Check Names
> OK
> OK
> OK
```

| Location | Label | Letter | Target group |
| --- | --- | --- | --- |
| `\\Techworks-DC01\Finance` | Finance | `F` | `GG_Finance` |
| `\\Techworks-DC01\GIS` | GIS | `M` | `GG_GIS` |
| `\\Techworks-DC01\Operations` | Operations | `O` | `GG_Operations` |
| `\\Techworks-DC01\GIS` | GIS | `M` | `GG_Operations` |

**Expected result:** each user sees only their own department drive in This PC.

**Notes:**

- `G:` and `H:` were already used by the existing `GPO - Drive mapping`, so GIS uses `M:`. Check existing letters before choosing.
- The yellow triangle on each item is the icon for the **Update** action, not a warning.
- Two items can share a letter when item-level targeting ensures a user only matches one. One item with two Security Group conditions joined by **OR** (Item Options > Or) does the same job.

> **Roadblock: the drive did not map, with no error logged.**
>
> **Symptom:** no `F:` for Amy, though she could open `\\Techworks-DC01\Finance` by path. The Application log had no "Group Policy Drive Maps" events.
>
> **Cause:** `whoami /groups` showed no `GG_` groups in her token. With auto logon, Windows signed her in before the network was ready, using cached credentials that carried her old group list. Item-level targeting checked the token, found no match, and skipped the item.
>
> **Fix:** enable **Always wait for the network at computer startup and logon** (A3), then restart the client.

---

## Part F: File activity in the user script

**Outcome:** after each typing session, the user performs one random action on their department share and logs it.

| Action | What it does | Server evidence |
| --- | --- | --- |
| `FILE_CREATE` | Creates `note-<timestamp>.txt` | 4663 |
| `FILE_READ` | Reads a random file | 4663 |
| `FILE_MODIFY` | Appends a line to a random file | 4663 |
| `OTHER_DEPT_ATTEMPT` | Tries to list another department's share | 4656 Audit Failure for Amy and Ben; succeeds for Cara on GIS |

Each action writes a line to `Documents\simulation.log`:

```
2026-11-16 15:34:25 | TW-CL00 | amy.fin | FILE_READ | SUCCESS
```

**Why this runs as the user and not as SYSTEM:** `SYSTEM` has no mapped drives, is in no department group, and appears in the audit log as the computer account. Activity is only attributable to a person if a process running as that person performs it.

---

## 6. Verification

**Daily check (after 8:05 AM on a weekday).** On the VMware host:

```powershell
& "C:\Program Files\VMware\VMware Workstation\vmrun.exe" list
schtasks /Query /TN "Lab\Clients-Start" /V /FO LIST | findstr /C:"Last Run Time" /C:"Last Result"
```

On the DC:

```powershell
Invoke-Command -ComputerName TW-CL00, TW-CL01, TW-CL02 -ScriptBlock {
    [pscustomobject]@{ SignedInUser = (quser)[1].Trim().Split()[0] }
} | Select-Object PSComputerName, SignedInUser
```

**Finding the evidence in Event Viewer** (on the file server):

```
Event Viewer
> Windows Logs
> Security
> Filter Current Log
> Logged: Last hour
> Event IDs: 4663
> OK
> Find
> type the user name or file name
> Find Next
```

To return to the full log, click **Clear Filter** in the Actions pane. A filter hides events; it never deletes them.

**Saved view for refusals:**

```
Event Viewer
> right-click Custom Views
> Create Custom View
> Logged: Last 24 hours
> By log: Windows Logs > Security
> Event IDs: 4656,5145
> Keywords: tick Audit Failure
> OK
> Name: Sim - File Access Denied
> OK
```

| Event ID | Meaning | Key fields |
| --- | --- | --- |
| 4663 | A file or folder was accessed | Account Name, Object Name, Accesses |
| 4656 (Audit Failure) | A request to open a file or folder was refused | Account Name, Object Name, Access Reasons |
| 5145 | A share-level access check | Account Name, Share Name, Relative Target Name, Source Address |
| 4098 (Application log) | A Group Policy Preference item failed | Item name, GPO name, error code |

## 7. Rollback

| To undo | Do this |
| --- | --- |
| Scheduled power on/off | `schtasks /Delete /TN "Lab\Clients-Start" /F` and the same for `Lab\Clients-Stop` |
| Auto logon | Run Autologon on the client and click **Disable** |
| Activity task | `Unregister-ScheduledTask -TaskName 'Sim Typing' -Confirm:$false` on each client. To pause only: `Disable-ScheduledTask -TaskName 'Sim Typing'` |
| A GPO's effect | Remove the link from the OU (right-click the link > Delete). The GPO itself is kept |
| Shares | `Remove-SmbShare -Name Finance`. The folder and its data remain |
| Groups and users | Delete from `OU=Sim` in Active Directory Users and Computers |

## 8. Troubleshooting

| Symptom | Likely cause | Check | Fix |
| --- | --- | --- | --- |
| GPO not listed in `gpresult` | Object is not in the OU the GPO is linked to, or the wrong scope was checked | `gpresult /r /scope computer` (elevated) and `/scope user` (as the user). Read the `CN=...,OU=...` line | Link the GPO to the right OU |
| Setting applied but the wrong value wins | Another GPO sets the same thing | `gpresult /h C:\gp.html /scope computer /f`, Winning GPO column | Correct the setting; Policies override Preferences |
| User added to a group but still has no drive or access | Group membership is only read at sign-in; or the sign-in used cached credentials | `whoami /groups` | Sign out and back in; enable "Always wait for the network" |
| Preference item (task, drive) missing, `gpupdate` says success | The item failed or was filtered out | Client Application log, event 4098; for drive maps, `whoami /groups` | Depends on the error code (see Part D, Part E) |
| Script "is not digitally signed" | Execution policy from GPO; path treated as Internet zone | `Get-ExecutionPolicy -List` | Use `\\TECHWORKS\...` instead of `\\techworks.local\...` |
| Remote command fails with `0x8009030e` | Run from a machine or account with no domain identity | `hostname`, `whoami` | Run from a domain-joined machine as a domain account |
| Admin refused on a folder; file will not save to NETLOGON | UAC has switched off the admin groups in a normal window | `whoami /groups \| findstr /i "admin"` shows "deny only" | Elevate, or use a dedicated admin group |
| No Audit Failure events for refused folder access | Handle Manipulation auditing is off | `auditpol /get /subcategory:"Handle Manipulation"` | Enable Failure in `Sim - Audit Policy` |
| Custom view shows successes too | The Audit Failure keyword was not saved | View Properties > Edit Filter | Tick Audit Failure |
| Typing appears in the wrong window during testing | The script sends keys to whichever window is in front | n/a | `Disable-ScheduledTask` on that client while testing, then re-enable |
| Task not visible in Task Scheduler | A subfolder is selected, or Task Scheduler is not elevated | Click **Task Scheduler Library** itself | Run Task Scheduler as administrator |
| File saved as `name.txt.txt` | Windows hides extensions; the user typed one as well | The audit log's Object Name shows the true name | Rename; consider showing extensions |
| Copy and paste between host and VM fails | VMware Tools missing | Look for the VMware Tools service in the guest | VM > Install VMware Tools |

## 9. Key lessons

> ### After a group change, the user must sign out and back in, and that sign-in must reach a domain controller. `whoami /groups` proves it in ten seconds.
>
> Windows reads a user's group memberships once, at sign-in, and stores them in the session's token. Adding someone to a group changes Active Directory immediately but does not change a session that is already running. If the sign-in happens before the network is ready, Windows uses cached credentials with the old group list. This is behind most "I was added to the group but still can't get in" tickets.

Other lessons from this build:

- **Scope first.** For any "policy isn't applying" problem, ask whether it is a computer or a user setting, and whether that object sits in the OU the GPO is linked to.
- **`gpupdate` success is not proof.** Check the result (`gpresult`, the task, the drive) and the client's Application log.
- **Know where you are.** Run `hostname` and `whoami` before changing other machines. Two PowerShell windows look identical.
- **Elevated or not matters.** An administrator's normal window is not an administrator. This affects file access, saving to system shares, and `schtasks`.
- **Read the audit log as evidence.** A script's own log is a claim. The server's Security log is independent proof.
- **Variables do not cross scripts.** Each script runs in its own session.
- **Verify every edit.** `type` or `Get-Content` the file after saving.

## 10. Command reference

| Task | Command |
| --- | --- |
| Where am I, who am I | `hostname` / `whoami` |
| Groups in my session | `whoami /groups` |
| Applied computer policy (elevated) | `gpresult /r /scope computer` |
| Applied user policy (as the user) | `gpresult /r /scope user` |
| Full policy report with winning GPOs | `gpresult /h C:\gp.html /scope computer /f` |
| Refresh policy | `gpupdate /force` |
| Effective audit policy | `auditpol /get /category:*` |
| One audit subcategory | `auditpol /get /subcategory:"Handle Manipulation"` |
| Execution policy by scope | `Get-ExecutionPolicy -List` |
| Start / stop a VM | `& $vmrun -T ws -vp $pw start <vmx> nogui` / `... stop <vmx> soft` |
| Running VMs | `& "C:\Program Files\VMware\VMware Workstation\vmrun.exe" list` |
| Find a program | `where /R "C:\Program Files" vmrun.exe` |
| Create a weekday task | `schtasks /Create /TN "<name>" /SC WEEKLY /D MON,TUE,WED,THU,FRI /ST 08:00 /TR "<command>"` |
| Run / query a task | `schtasks /Run /TN "<name>"` / `schtasks /Query /TN "<name>" /V /FO LIST` |
| Pause / resume a task | `Disable-ScheduledTask -TaskName '<name>'` / `Enable-ScheduledTask -TaskName '<name>'` |
| Run a command on a client | `Invoke-Command -ComputerName <pc> -ScriptBlock { ... }` |
| Who is signed in | `quser` |
| Restart a client and wait | `Restart-Computer -ComputerName <pc> -Force -Wait -For PowerShell` |
| Network drives in a session | `net use` |
| Shares on a server | `Get-SmbShare` |
| Folder permissions | `icacls <folder>` |
| Where a computer or user sits | `Get-ADComputer <name>` / `Get-ADUser <name>` (read `DistinguishedName`) |
| Is a setting in a GPO | `(Get-GPOReport -Name '<gpo>' -ReportType Xml) -match '<text>'` |
| Recent preference failures | `Get-WinEvent -FilterHashtable @{LogName='Application'; ProviderName='Group Policy Scheduled Tasks'} -MaxEvents 5` |
| Search a file for text | `Select-String -Path <file> -Pattern '<text>'` |
| Print a file | `type <file>` |

## 11. Design notes

- **File server on a domain controller.** Done because the lab has only two Windows servers. In production, file services belong on a member server with a dedicated data volume.
- **GPOs linked to `Workstations`.** The Sim computer settings apply to every computer in that OU. Acceptable while all three are lab clients. A dedicated OU or security filtering would be needed if other machines join it.
- **Auto logon.** It stores a password on each client and removes the interactive sign-in. Lab only. The accounts are standard users with access to test data only.
- **Task delivery by remoting, not Group Policy.** The tasks exist per machine and are not re-applied automatically. A rebuilt or new client needs the `Register-ScheduledTask` block run again. A machine startup script or a configuration management tool would close that gap.
- **Hard-coded user list.** The script names its three users and maps each to a department. Deriving both from group membership would scale better.
- **VM password at rest.** DPAPI ties `vmpw.txt` to one Windows account on one PC. Changing the account or the VM password requires re-creating the file.
- **Host dependency.** The schedule depends on a workstation being on with a user signed in. Moving the clients to Proxmox would allow an always-on scheduler (`qm start` / `qm shutdown` from cron).
- **Audit volume.** One file save produces several 4663 events. On a busy server this fills the Security log quickly; log size and forwarding should be planned.
- **SendKeys.** Typing goes to the foreground window, so the sessions cannot be used interactively while the script runs. A purpose-built simulator such as GHOSTS (Carnegie Mellon SEI) avoids this.

## 12. Open items

- [ ] Confirm the account lockout policy (threshold 5, duration 15 minutes) in the Default Domain Policy.
- [ ] Check TW-CL01 and TW-CL02 for the older `TechWorks-UserSimulation` task.
- [ ] Set Notepad on each client to start a new session (Settings > When Notepad starts), so unsaved tabs do not pile up.
- [ ] Remove the unused `Sim\Computers` OU, or move the clients into it and relink.
- [ ] Create a separate admin account for daily administration and add it to `DL_Shares_Admin`.
- [ ] Next phases: accounting application, GIS application, lab email, ticket system with fault injection, Windows Event Forwarding.

---

## Appendix A: Invoke-SimTyping.ps1

Location: `\\TECHWORKS\NETLOGON\Sim\Invoke-SimTyping.ps1`

```powershell
param([int]$Minutes = 0)
if ($env:USERNAME -notin 'amy.fin','ben.gis','cara.ops') { return }
# --- File activity on the department share ---
$dept    = @{ 'amy.fin'='Finance'; 'ben.gis'='GIS'; 'cara.ops'='Operations' }[$env:USERNAME]
$share   = "\\Techworks-DC01\$dept"
$others  = 'Finance','GIS','Operations' | Where-Object { $_ -ne $dept }
$logFile = "$env:USERPROFILE\Documents\simulation.log"

function Write-SimLog($Action, $Result) {
    "{0} | {1} | {2} | {3} | {4}" -f (Get-Date -Format 'yyyy-MM-dd HH:mm:ss'), $env:COMPUTERNAME, $env:USERNAME, $Action, $Result |
        Add-Content $logFile
}

function Invoke-FileActivity {
    $action = 'FILE_CREATE','FILE_READ','FILE_MODIFY','FILE_MODIFY','OTHER_DEPT_ATTEMPT' | Get-Random
    try {
        switch ($action) {
            'FILE_CREATE' { "Work note created $(Get-Date)" | Set-Content "$share\note-$(Get-Date -Format 'yyyyMMdd-HHmmss').txt" -ErrorAction Stop }
            'FILE_READ'   { $f = Get-ChildItem $share -File -ErrorAction Stop | Get-Random; Get-Content $f.FullName -ErrorAction Stop | Out-Null }
            'FILE_MODIFY' { $f = Get-ChildItem $share -File -ErrorAction Stop | Get-Random; "Updated $(Get-Date)" | Add-Content $f.FullName -ErrorAction Stop }
            'OTHER_DEPT_ATTEMPT' { $t = $others | Get-Random; Get-ChildItem "\\Techworks-DC01\$t" -ErrorAction Stop | Out-Null }
        }
        Write-SimLog $action 'SUCCESS'
    }
    catch { Write-SimLog $action "FAILED: $($_.Exception.Message)" }
}
# --- End file activity block ---

# Run until 11:50 AM by default, or for a set number of minutes when testing
if ($Minutes -gt 0) { $end = (Get-Date).AddMinutes($Minutes) }
else                { $end = (Get-Date).Date.AddHours(11).AddMinutes(50) }

$words = 'invoice','quarterly','meeting','budget','review','client',
         'update','schedule','parcel','report','vendor','follow','up'
$wsh = New-Object -ComObject WScript.Shell

while ((Get-Date) -lt $end) {
    Start-Process notepad
    Start-Sleep -Seconds 4
    $null = $wsh.AppActivate('Notepad')

    1..(Get-Random -Minimum 3 -Maximum 9) | ForEach-Object {
        $line = ($words | Get-Random -Count 5) -join ' '
        foreach ($c in $line.ToCharArray()) {
            $wsh.SendKeys([string]$c)
            Start-Sleep -Milliseconds (Get-Random -Minimum 60 -Maximum 280)
        }
        $wsh.SendKeys('{ENTER}')
        Start-Sleep -Seconds (Get-Random -Minimum 5 -Maximum 40)
    }

    Stop-Process -Name notepad -Force -ErrorAction SilentlyContinue
    Invoke-FileActivity
    Start-Sleep -Seconds (Get-Random -Minimum 300 -Maximum 1200)   # idle 5-20 min
}
```

## Appendix B: Document history

| Date | Change | By |
| --- | --- | --- |
| 2026-11-16 | First issue. Covers AD groundwork, power schedule, auto logon, activity script, file shares and auditing | Techworks |
