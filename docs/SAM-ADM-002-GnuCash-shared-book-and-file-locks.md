# SAM-ADM-002: GnuCash shared book and file locks

| Field | Value |
| --- | --- |
| Document ID | SAM-ADM-002 |
| Title | Deploying GnuCash with a company book on a department share, and resolving "file is locked" tickets |
| Owner / Author | Techworks |
| Status | Built and verified on `TW-CL00` on 2026-11-16. See Open Items for what was not confirmed |
| Last updated | 2026-11-16 |
| Depends on | SAM-ADM-001 (Finance share, security groups, drive map, auditing) |
| Companion file | `techworks-sample-transactions.csv` (sample data, 125 transactions) |

**Search keywords:** `GnuCash`, `could not obtain the lock`, `.LCK`, `stale lock`, `file is locked by another user`, `Open Anyway`, `Open Files`, `Sessions`, `4663`, `DELETE`, `import transactions CSV`

## How to read this document

| If you are | Read |
| --- | --- |
| Level 1 / Level 2 technician | Purpose, Prerequisites, the **Steps** and **Expected result** in each Part, Troubleshooting |
| Systems administrator / engineer | All of the above, plus the **Why** notes, Rollback, and Key Lessons |
| Architect / reviewer | Purpose, Design Notes, Open Items |

Conventions:

- Click paths are written one click per line, each starting with `>`.
- Every procedure states an **Expected result**. If you do not see it, stop and go to Troubleshooting.
- **Roadblock** notes record problems actually hit while building this, with cause and fix.

---

## 1. Purpose and outcome

Give the Finance user a real desktop accounting application whose data file lives on the department share, then reproduce, investigate and resolve the most common support call for that kind of software: "it says the company file is locked by another user".

**You know it works when:**

- GnuCash opens `F:\Books\techworks-company.gnucash` as the Finance user with no warning.
- The Checking Account total reads $114,446.22.
- You can produce the lock warning on demand, decide from evidence whether the lock is stale, clear it, and find the record of that in the server's Security log.

## 2. Scope and environment

| Item | Value |
| --- | --- |
| Client | `TW-CL00` (Windows 11) |
| User | `amy.fin` (standard user, member of `GG_Finance`) |
| File server | `Techworks-DC01` |
| Share and folder | `\\Techworks-DC01\Finance`, mapped as `F:`; server path `C:\Shares\Finance\Books` |
| Application | GnuCash (current Windows release from gnucash.org) |
| Book | `techworks-company.gnucash`, XML format |
| Fictional company | Techworks Supply Co |

Out of scope: a second Finance user, GnuCash on macOS, and web-based accounting (see Open Items).

## 3. Background: how GnuCash handles its data file

| Fact | Why it matters to support |
| --- | --- |
| The whole company is one file on the share | Same arrangement as other desktop accounting products. Permissions, drive maps and network problems all show up as application problems |
| Only one person can have a book open for editing | A second person gets the lock warning |
| While a book is open, an empty file named `<book>.LCK` sits beside it | That file **is** the lock. If it exists, GnuCash assumes someone is in the book |
| GnuCash removes the `.LCK` file when it closes normally | A lock left behind after an abnormal exit is a **stale lock** |
| GnuCash reads the book into memory and then releases the file | The book will usually **not** appear under Open Files on the server even while someone is working in it |
| Each save creates a dated backup (`<book>.<date-time>.gnucash`) and a change log (`.log`) | These are how you recover from "I deleted something and saved". Retention is set in GnuCash Preferences; the folder grows over time |

## 4. Prerequisites

- SAM-ADM-001 Part E complete: `F:` maps for the Finance user, and auditing is on for `C:\Shares\Finance`.
- A domain admin username and password, to authorize the install.
- Internet access from the client to download the installer.

---

## Part A: Install GnuCash

**Outcome:** GnuCash is installed on the client and has not yet been run.

**Steps** (on `TW-CL00`, signed in as the Finance user):

```
Microsoft Edge
> address bar: https://www.gnucash.org/download.phtml
> under Microsoft Windows, click the installer link (gnucash-<version>.setup.exe)
> wait for the download to finish
```

```
File Explorer
> Downloads
> right-click gnucash-<version>.setup.exe
> Run as administrator
> enter a domain admin username and password
> Yes
```

```
Select Setup Language: English
> OK
> Next
> I accept the agreement
> Next
> Next (default install folder)
> Next (default components)
> Next
> Install
> untick "Run GnuCash now"
> Finish
```

**Expected result:** typing `GnuCash` in the Start menu lists the application.

**Why:** a standard user cannot install software, which is correct. The technician supplies admin credentials for the installer only. Untick "Run GnuCash now" because a program launched by the installer runs as the administrator account, and its first-run settings would be saved to the wrong user profile.

## Part B: Create the company book on the share

**Outcome:** a new book with a business chart of accounts, saved to the Finance share.

**Steps** (as the Finance user):

```
File Explorer
> This PC
> Finance (F:)
> right-click an empty area
> New
> Folder
> name it: Books
```

```
Start
> type: GnuCash
> click GnuCash
> close the Tip of the Day window if it appears
> Welcome to GnuCash: select "Create a new set of accounts"
> OK
```

```
New Account Hierarchy Setup
> Next
> New Book Options: Business tab
> Company Name: Techworks Supply Co
> Next
> Choose Currency: US Dollar (USD)
> Next
> Choose accounts to create
> untick Common Accounts
> tick Business Accounts
> Next
> Setup selected accounts
> Next
> Finish Account Setup
> Apply
```

```
File
> Save As...
> Data Format: XML
> navigate to: Finance (F:) > Books
> Name: techworks-company
> Save As
```

**Expected result:**

- Title bar reads `techworks-company.gnucash - Accounts - GnuCash`.
- The Accounts tab lists Assets, Equity, Expenses, Income and Liabilities.
- `F:\Books` contains `techworks-company.gnucash` and, while GnuCash is open, `techworks-company.gnucash.LCK`.

**Why XML:** it keeps the whole company in a single file on the share, which is what produces the locking and permission scenarios this document is about.

## Part C: Load the sample data

**Outcome:** three months of realistic activity in the book.

The companion file `techworks-sample-transactions.csv` holds 125 transactions for 2026-08-01 to 2026-10-31. All company and customer names are invented. Its account names match the Business Accounts set exactly.

| Type | Contents |
| --- | --- |
| Opening balances | $42,500 checking, $15,000 savings |
| Sales | 12 deposits a month, as invoices to ten fictional customers |
| Monthly bills | Rent, electric, water, internet, phone, liability insurance, janitorial, bookkeeping, bank charge |
| Payroll | Twice a month, with the employer tax entry |
| Credit card | Supplies, fuel, a subscription, meals, repairs, printing, and a monthly card payment |
| Petty cash | Postage, courier, parking, coffee |
| Savings | A monthly transfer in and an interest entry |

**Steps:**

Copy the CSV into `F:\Books` on the client, then in GnuCash:

```
File
> Import
> Import Transactions from CSV...
> Next
> browse to Finance (F:) > Books
> select techworks-sample-transactions.csv
> Next
```

On the **Import Preview** page:

```
File Format
> Character-separated
> tick Comma (untick the others)

Miscellaneous
> Date Format: y-m-d
> Leading Lines to Skip: 1
> leave Multi-split unticked
```

Set the drop-down above each column:

| Column contents | Drop-down |
| --- | --- |
| `2026-08-01` | Date |
| blank or `2102` | Number |
| `Opening balance - checking` | Description |
| `Assets:Current Assets:Checking Account` | Account |
| `42500.00` | Amount |
| `Equity:Opening Balances` | Transfer Account |

**Checkpoint:** the heading row is shown struck through, and no rows are highlighted as errors.

```
Import Preview
> Next
> Match Import and GnuCash accounts: confirm every row has an account on the right
> Next
> Transaction Information
> Next
> Match Transactions: confirm rows are marked to add
> Next
> Import Summary
> Close
```

```
File
> Save
```

**Expected result:** the asterisk disappears from the title bar, and the Accounts tab shows:

| Account | Total |
| --- | --- |
| Checking Account | $114,446.22 |
| Savings Account | $18,014.88 |
| Petty Cash | $289.66 |
| Credit Card | $322.49 |
| Sales | $133,751.39 |
| Rent | $7,200.00 |
| Payroll Expenses | $41,279.69 |

These totals were calculated from the file before import, so a match confirms all 125 transactions loaded correctly.

**Why:** in the CSV, a positive amount is money into the account in the Account column and a negative amount is money out. Accounts Receivable and Accounts Payable are not used, because GnuCash reserves those for its invoicing features.

> **Roadblock:** after the import the title bar showed `*techworks-company.gnucash`. The asterisk means unsaved changes: the data was in memory, not yet on the share. **Fix:** File > Save.

---

## Part D: Reproduce the "file is locked" warning

**Outcome:** the lock warning appears on demand.

**Steps:**

```
GnuCash
> File
> Quit
```

```
File Explorer
> Finance (F:)
> Books
> press F5
> confirm there is no .LCK file
```

```
Start
> type: cmd
> press Enter
```

```
type nul > "F:\Books\techworks-company.gnucash.LCK"
```

```
Start
> type: GnuCash
> click GnuCash
```

**Expected result:** a dialog reading:

```
GnuCash could not obtain the lock for F:\Books\techworks-company.gnucash.
That database may be in use by another user, in which case you should not open the database. What would you like to do?
```

with the buttons **Open Read-Only**, **Create New File**, **Open Anyway**, **Open Folder**, **Quit**.

**Why this method:** a stale lock is only an empty file with that name. Creating it directly gives the same result as a crash and works every time.

> **Roadblock:** two attempts to create a stale lock by killing GnuCash did not produce the warning.
>
> | Attempt | Result | Cause |
> | --- | --- | --- |
> | Task Manager > Processes > End task | Lock file removed; book reopened normally | End task on the Processes tab asks the program to close, the same as clicking X. GnuCash shut down cleanly and removed its lock |
> | Task Manager > Details > `gnucash.exe` > End task > End process | Book reopened with no warning | Not determined. It was not confirmed whether the lock file remained after the kill |
>
> **Fix:** create the lock file directly, as above.
>
> Use `type nul >` and not right-click > New > Text Document. Windows hides file extensions, so the second method produces `...LCK.txt`, which GnuCash ignores.

## Part E: Investigate before touching anything

**Outcome:** a decision, based on evidence, about whether someone is really in the book.

What each button does:

| Button | Effect | Risk |
| --- | --- | --- |
| Open Read-Only | View only; nothing can be saved | None |
| Create New File | Abandons this book for an empty one | The user believes their data is gone |
| Open Anyway | Ignores the lock and opens for editing | If someone else is in the book, the two overwrite each other |
| Open Folder | Shows the folder | None |
| Quit | Closes GnuCash | None |

Leave the dialog open and gather three pieces of evidence on the file server.

**Evidence 1: who is connected, and what is open**

```
Server Manager
> Tools
> Computer Management
> System Tools
> Shared Folders
> Sessions
```

```
Shared Folders
> Open Files
```

**Evidence 2: who created the lock, and when**

```
File Explorer (on the server)
> C:\Shares\Finance\Books
> right-click techworks-company.gnucash.LCK
> Properties
> General tab: note Created
> Security tab
> Advanced
> read the Owner line
```

**Evidence 3: the audit trail**

```
Event Viewer
> Windows Logs
> Security
> Filter Current Log...
> Logged: Last hour
> Event IDs: 4663
> leave User as <All Users>
> OK
> Find...
> type: LCK
> Find Next
```

**What was found on 2026-11-16:**

| Evidence | Finding | Meaning |
| --- | --- | --- |
| Sessions | One connection: `amy.fin` from `TW-CL00` | Nobody else was connected to the server |
| Open Files | `C:\Shares\Finance\Books`, `amy.fin`, mode Read | The folder, held by her File Explorer window. Neither the book nor the lock was open |
| Owner of `.LCK` | `amy.fin` | Her own session created the lock |
| Audit log | Event 4663 for the `.LCK` file | Confirms the account and time |

**How to decide:**

| If | Then |
| --- | --- |
| The only session is the person reporting the problem, and they own the lock | The lock is stale. Clear it (Part F) |
| Another Finance user has a session, or owns the lock | Contact that person and ask before doing anything. If they are in the book, the lock is doing its job |
| The lock's owner has no session and cannot be reached | Use **Open Read-Only** for the user for now, and escalate |

**Why Open Files is not enough on its own:** GnuCash loads the book into memory and releases the file, so the book not being listed does not prove nobody is using it. Sessions and the lock's owner are the stronger evidence.

## Part F: Clear a stale lock

**Outcome:** the book opens normally and the action is recorded.

**Steps** (on the client):

```
GnuCash warning
> Quit
```

```
File Explorer
> Finance (F:)
> Books
> right-click techworks-company.gnucash.LCK
> Delete
```

Delete only the file ending in `.LCK`. The file ending in `.gnucash` is the company's books.

```
Start
> type: GnuCash
> click GnuCash
```

**Expected result:** the book opens with no warning, Checking Account reads $114,446.22, and a new `.LCK` file appears in the folder.

**Why Quit and delete, not Open Anyway:** both work on a stale lock. Deleting one named file after checking the evidence is a deliberate act. Users who are taught to click Open Anyway will click it every time, including when a colleague really is in the book.

**Verify in the audit log** (on the server), using the filter from Part E. Look for:

```
Account Name:   amy.fin
Object Name:    C:\Shares\Finance\Books\techworks-company.gnucash.LCK
Accesses:       DELETE
```

---

## 6. Rollback

| To undo | Do this |
| --- | --- |
| A bad import or unwanted edits | Close GnuCash. In `F:\Books`, copy the most recent good `techworks-company.gnucash.<date-time>.gnucash` backup over `techworks-company.gnucash` |
| The whole book | Delete the contents of `F:\Books` |
| The application | Settings > Apps > Installed apps > GnuCash > Uninstall (admin credentials needed) |

## 7. Troubleshooting

| Symptom | Likely cause | Check | Fix |
| --- | --- | --- | --- |
| "GnuCash could not obtain the lock" | Someone is in the book, or a stale `.LCK` file | Part E | Part F if stale; otherwise contact the other user |
| Title bar shows `*` before the file name | Unsaved changes | n/a | File > Save |
| Import stops at "Match Import and GnuCash accounts" with an empty row | An account name in the CSV is not in the book | Compare spelling with the Accounts tab | Double-click the row and choose the account |
| Totals do not match after import | Wrong date format, heading row not skipped, or a column mapped wrongly | Re-open the import and check the Preview page | Restore the backup (Rollback) and import again |
| Rows shown in red on the Import Preview page | A value cannot be read with the chosen formats | Date Format and separator settings | Correct them before clicking Next |
| Book opens read-only or will not save | The user has Read but not Modify on the folder | `whoami /groups` for `DL_Finance_RW` | Fix group membership, then sign out and back in (SAM-ADM-001) |
| `F:` drive missing | Drive map did not apply | SAM-ADM-001 Part E | SAM-ADM-001 Part E |
| Event Viewer filter returns 0 events | A name was typed into the **User** box | Read the filter line above the list | Clear the User box; use Find for the name |
| A file planted as a lock is ignored | It was created as `...LCK.txt` | Turn on file extensions in File Explorer | Create it with `type nul >` |

## 8. Key lessons

> ### Before clearing a lock, prove who holds it. Sessions shows who is connected, the lock file's Owner shows who created it, and the audit log shows when.

- **End task is not a crash.** On Task Manager's Processes tab it asks the program to close. A forced kill is done from the Details tab.
- **The User box in an Event Viewer filter does not work for Security events.** Their User field is always `N/A`. The account is inside the message, as Account Name. Filter by event ID and time, then use Find for the name.
- **Your own investigation is logged.** Opening a file's Properties on the server produced a 4663 event (`READ_CONTROL`, process `explorer.exe`) under the investigator's account.
- **Identical events can mean different things.** Two `DELETE` events on the `.LCK` file by the same user were recorded: one when GnuCash quit normally, one when a technician removed the stale lock. Only the timestamps, matched against what was happening, tell them apart.
- **`Process ID: 0x4` with a blank Process Name** means the action arrived over the network through file sharing. A named process such as `explorer.exe` means it was done at the server itself.
- **Do not launch a program from its installer.** It runs as the installing administrator, not the user.
- **An asterisk in the title bar means not saved.**

## 9. Command and path reference

| Task | Command or path |
| --- | --- |
| Create an empty file with an exact name | `type nul > "F:\Books\techworks-company.gnucash.LCK"` |
| Who is connected to the server | Computer Management > System Tools > Shared Folders > Sessions |
| Which files are open through shares | Computer Management > System Tools > Shared Folders > Open Files |
| Same, from PowerShell | `Get-SmbSession` / `Get-SmbOpenFile` |
| Who created a file | Properties > Security > Advanced > Owner |
| Groups in a user's session | `whoami /groups` |
| File access events | Event Viewer > Security, Event ID `4663`, then Find |
| Force-kill a process | Task Manager > Details > right-click the process > End task, or `taskkill /F /IM gnucash.exe` |

## 10. Design notes

- **Single-user file on a share.** This is the weakest design for shared accounting data: one editor at a time, and the lock is advisory, relying on every client honoring a marker file. It was chosen because it reproduces the support problems of common desktop accounting products. A web application with a database (planned next) removes the lock problem and introduces others: server uptime, database backup, and login integration.
- **Backups inside the data folder.** GnuCash's automatic backups sit beside the live book on the same volume. They protect against user error, not against loss of the server. Server-level backup of `C:\Shares` is still required.
- **Sensitive data location.** A real company book contains payroll and banking detail. Access here is limited to `DL_Finance_RW`, `DL_Finance_RO` and `DL_Shares_Admin`, with all access audited.
- **Audit volume.** Each save writes the book, a backup and a log, producing several 4663 events. On a busy share this needs log sizing or forwarding.
- **Sample data in a share.** The CSV was imported from `F:\Books`. Import source files should be removed from the data folder afterwards.

## 11. Open items

- [ ] Find and record the second `DELETE` event (the manual removal at about 5:00 PM). Only the 4:47:03 PM event, from GnuCash quitting normally, was captured.
- [ ] Determine why a forced kill from the Details tab did not leave a lock that triggered the warning.
- [ ] Remove `techworks-sample-transactions.csv` from `F:\Books`.
- [ ] Add a second Finance user, to produce a real two-person lock conflict.
- [ ] Test GnuCash on the domain-joined MacBook (macOS High Sierra). Confirm which GnuCash version supports that macOS before installing.
- [ ] Add opening GnuCash to the Finance user's activity script (SAM-ADM-001), so the book is touched every morning.
- [ ] Next stage: web-based accounting (Odoo Community), per SAM-015.

## Document history

| Date | Change | By |
| --- | --- | --- |
| 2026-11-16 | First issue | Techworks |
