# CLAUDE.md

Instructions for Claude Code when working in this repository. Read this before making any change.

## What this repository is

Build documentation, scripts and sample data for the Techworks Enterprise Lab: a home lab run like a small company, with simulated employees, Active Directory, file shares with auditing, business applications and security monitoring. It is a public portfolio. Everything committed here can be read by anyone.

The owner is building hands-on experience for a systems administration career. The point of the lab is that they do the work themselves.

## How to work with the owner

- **Explain, then propose.** For any change to a lab system (servers, Active Directory, GPOs, firewalls, VMs), explain what to do and why, and let the owner carry it out. Do not run commands that change lab systems unless the owner explicitly asks you to.
- **Manual method first.** When describing a task, give the GUI or hand-typed method first, then the scripted equivalent.
- **Click paths one click per line**, each starting with `>`:

  ```
  Group Policy Management
  > right-click Sim - Audit Policy
  > Edit
  > Computer Configuration
  ```

- **Exact names.** Give the exact setting name, menu item and value. Never write a vague step such as "configure the policy".
- **One step at a time** when walking through something new, with the expected result after each step, so problems are caught where they happen.
- **Before suggesting a command that changes other machines,** remind the owner to check `hostname` and `whoami` first.

## Documentation standard

Every document follows the same layered structure, so that a Level 1 technician and an architect can both use it without the content being simplified:

1. Header table: Document ID, Title, Owner / Author, Status, Last updated, Depends on
2. **Search keywords:** the exact error strings and terms someone would search for
3. How to read this document (which sections suit which reader)
4. Purpose and outcome, with a "You know it works when" test
5. Scope and environment
6. Prerequisites
7. Procedure in Parts: numbered steps, each with an **Expected result**, a **Why** note, and an **Automation equivalent** after the manual method
8. **Roadblock** notes where problems were actually hit: symptom, cause, proof, fix
9. Verification
10. Rollback
11. Troubleshooting table: Symptom, Likely cause, Check, Fix
12. Key lessons
13. Command reference
14. Design notes
15. Open items (unverified steps are listed here, never presented as tested)
16. Document history

Rules:

- **Owner / Author is always `Techworks`.** Never a personal name.
- Write in plain, direct sentences. Use tables for comparisons and reference material.
- Never present a step as verified unless it was run in the lab. Mark untested material as untested.

## File naming

| Series | Pattern | Example | Used for |
| --- | --- | --- | --- |
| SAM | `SAM-<nnn>-<Hyphenated-Description>.md` | `SAM-016-NO_PUBKEY-error.md` | Administration plans and incident write-ups |
| SAM-ADM | `SAM-ADM-<nnn>-<Hyphenated-Description>.md` | `SAM-ADM-002-GnuCash-shared-book-and-file-locks.md` | The Windows administration lab build series |
| RB | `RB-<Platform>-<nnn>-<Hyphenated-Description>.md` | `RB-Proxmox-018-Fleet-Update-Management.md` | Runbooks, numbered per platform |

Numbers are three digits. Before creating a document, list the existing ones and use the next free number in that series. Confirm the number with the owner.

## Repository layout

```
docs/               SAM and SAM-ADM documents
runbooks/           RB runbooks
scripts/powershell/ Windows scripts
scripts/bash/       Linux scripts
data/               Sample data (fictional only)
diagrams/           Architecture and flow diagrams
tools/hooks/        Git hooks, including the sensitive-data check
```

## Keeping sensitive data out (mandatory)

This repository is public. Before every commit:

- **Never commit** passwords, password hashes, API keys, tokens, private keys, certificates, VPN configs, `vmpw.txt`, `.env` files, or exported credentials.
- **Never commit real network details.** Replace real IP addresses with RFC 5737 documentation addresses (`192.0.2.x`, `198.51.100.x`, `203.0.113.x`) or a placeholder such as `<DC01-IP>`.
- **Never commit personal account names or email addresses.** Use role names: `<admin-account>`, `<lab-host>`. The simulated users (`amy.fin`, `ben.gis`, `cara.ops`) are fictional and may stay.
- **Scrub screenshots** of IP addresses, hostnames of personal machines, and account names before adding them.
- Scripts must read secrets at run time (for example `Read-Host -AsSecureString`, or a DPAPI-protected file kept outside the repository). Never write a secret into a script.

The pre-commit hook in `tools/hooks/pre-commit` scans staged changes and blocks the commit if it finds one of these. It runs only after this is set once in each clone of the repository:

```
git config core.hooksPath tools/hooks
```

Personal strings to block (account names, real hostnames, email addresses) go one per line in `.git/info/sensitive-words`. That file lives inside `.git`, so it is never pushed. Never copy its contents into a tracked file.

If the hook blocks a commit, fix the content. Do not bypass it with `--no-verify`, and do not weaken its patterns.

## Commits

- One logical change per commit, with a message that says what changed, such as `Add SAM-ADM-003: RHEL joined to Active Directory`.
- Show the owner `git status` and `git diff --staged` before committing.
- Do not push without the owner's go-ahead.
