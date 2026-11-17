# SAM-016: apt NO_PUBKEY error on the HashiCorp repository

| Field | Value |
| --- | --- |
| Document ID | SAM-016 |
| Title | Fixing `NO_PUBKEY` on `apt-get update` after a vendor rotates its signing key |
| Owner / Author | Techworks |
| Status | Fix applied and verified on the Ubuntu DevOps server on 2026-11-16: `apt-get update` runs clean |
| Last updated | 2026-11-16 |
| Applies to | Ubuntu 24.04 (noble) with the HashiCorp apt repository. The method applies to any third-party apt repository |

**Search keywords:** `NO_PUBKEY`, `FC9CA96ACA026560`, `hashicorp`, `GPG error`, `signatures couldn't be verified`, `apt-get update`, `signed-by`, `keyring`

## How to read this document

| If you are | Read |
| --- | --- |
| Level 1 / Level 2 technician | Symptom, Procedure, Verification |
| Systems administrator / engineer | All of the above, plus Why, Rollback, Automation equivalent |
| Architect / reviewer | Cause, Design notes |

Every step gives the command, what it does, and the expected result. If you do not see the expected result, stop.

---

## 1. Purpose and outcome

Restore package updates from a third-party apt repository after its vendor replaces the key used to sign it.

**You know it works when** `sudo apt-get update` finishes with no `NO_PUBKEY` or `GPG error` lines for that repository.

## 2. Symptom

Running `sudo apt-get update` prints:

```
Err:1 https://apt.releases.hashicorp.com noble InRelease
  The following signatures couldn't be verified because the public key is not available: NO_PUBKEY FC9CA96ACA026560
W: An error occurred during the signature verification. The repository is not updated and the previous index files will be used.
W: Failed to fetch https://apt.releases.hashicorp.com/dists/noble/InRelease
W: Some index files failed to download. They have been ignored, or old ones used instead.
```

Other repositories update normally. Packages from the affected repository (Terraform, Vault, Packer and others) stop receiving updates.

## 3. Cause

apt only trusts a repository if its package index is signed by a key the server already holds. HashiCorp replaced its repository signing key on 2026-09-09. A server that added the repository before that date still holds the old key, so it cannot verify the new signature and skips the repository.

Nothing is broken on the server. Its copy of the vendor's public key is out of date.

| Item | Value |
| --- | --- |
| Key ID in the error | `FC9CA96ACA026560` |
| Full fingerprint of the new key | `D55C 0D1A C78A 8D81 26CB 631C FC9C A96A CA02 6560` |
| Key created | 2026-09-09, expires 2031-09-08 |
| Key owner | HashiCorp Security (HashiCorp Package Signing) |
| Where the vendor publishes it | `https://apt.releases.hashicorp.com/gpg`, fingerprint listed at `https://www.hashicorp.com/trust/security` |

The key ID in the error is the last 16 characters of the fingerprint.

## 4. Prerequisites

- `sudo` rights on the server.
- Outbound HTTPS access to the vendor's site.
- The vendor's published fingerprint, from the vendor's own website, to compare against.

## 5. Procedure

### Step 1: Find where the server keeps the key

```
grep -r hashicorp /etc/apt/sources.list.d/
```

**What it does:** searches the folder where extra package sources are defined. Read-only.

**Expected result:**

```
/etc/apt/sources.list.d/hashicorp.list:deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com noble main
```

The path after `signed-by=` is the key file to replace. Then check it exists and note its date:

```
ls -l /usr/share/keyrings/ | grep -i hashi
```

**Expected result:** one file. A date earlier than the vendor's rotation date confirms it is the old key. On the Ubuntu DevOps server it was dated Jul 5.

**Why:** the key location and format depend on how the repository was first added. Writing the new key anywhere other than the `signed-by` path leaves the error in place.

### Step 2: Back up the old key

```
sudo cp /usr/share/keyrings/hashicorp-archive-keyring.gpg /usr/share/keyrings/hashicorp-archive-keyring.gpg.bak
```

**Expected result:** no output. A `.bak` copy now exists beside the original.

### Step 3: Download the current key

```
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor --yes -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
```

| Piece | Meaning |
| --- | --- |
| `wget -O- <url>` | Download the key and pass it to the next command instead of saving it |
| `gpg --dearmor` | Convert it from text form to the binary form apt expects in this file |
| `--yes` | Allow overwriting the existing file |
| `-o <path>` | Write the result to the `signed-by` path from Step 1 |

**Expected result:** download progress, then a return to the prompt with no error.

### Step 4: Verify the fingerprint before trusting it

```
gpg --no-default-keyring --keyring /usr/share/keyrings/hashicorp-archive-keyring.gpg --fingerprint
```

**Expected result:**

```
pub   rsa4096 2026-09-09 [SC] [expires: 2031-09-08]
      D55C 0D1A C78A 8D81 26CB  631C FC9C A96A CA02 6560
uid           [ unknown] HashiCorp Security (HashiCorp Package Signing) <security+packaging@hashicorp.com>
```

Compare the fingerprint line with the value on the vendor's security page. They must match exactly.

Two harmless lines may appear the first time: `directory '/home/<user>/.gnupg' created` and `trustdb created`. `[ unknown]` beside the name is also normal; it refers to gpg's personal trust settings, not to apt.

**Why:** a signing key tells the server "packages from this source can be trusted". Installing one without checking it against the vendor's published fingerprint would let anyone who could tamper with the download have their packages trusted. **If the fingerprint differs, stop. Restore the backup (see Rollback) and do not run the update.**

### Step 5: Update the package lists

```
sudo apt-get update
```

**Expected result:** the HashiCorp line reads `Get:` or `Hit:`, and there are no `NO_PUBKEY`, `GPG error` or `Failed to fetch` lines.

## 6. Verification

```
sudo apt-get update 2>&1 | grep -i -E "hashicorp|NO_PUBKEY|GPG error"
```

**Pass:** only a `Hit:` or `Get:` line for `apt.releases.hashicorp.com`.

**Fail:** any `NO_PUBKEY` or `GPG error` line. Go to Troubleshooting.

Once it passes, the backup can be removed:

```
sudo rm /usr/share/keyrings/hashicorp-archive-keyring.gpg.bak
```

## 7. Rollback

Restore the previous key:

```
sudo cp /usr/share/keyrings/hashicorp-archive-keyring.gpg.bak /usr/share/keyrings/hashicorp-archive-keyring.gpg
```

This returns the server to its earlier state, including the original error.

## 8. Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| Same `NO_PUBKEY` error after the fix | The key was written to a different path than `signed-by` names | Re-run Step 1 and use that exact path in Step 3 |
| `NO_PUBKEY` with a different key ID | A different repository's key is out of date | Repeat this procedure for that vendor, using that vendor's key URL and published fingerprint |
| "Missing Signed-By in the sources.list entry" | The source line has no `signed-by=` option, or it is malformed | Edit the file in `/etc/apt/sources.list.d/` so the line includes `[signed-by=<path to key>]` |
| `wget` fails or times out | No outbound access, or a proxy or firewall is blocking the site | Test with `curl -I https://apt.releases.hashicorp.com/gpg`; check DNS and firewall rules |
| Fingerprint does not match the vendor's page | Wrong URL, a stale cached page, or a tampered download | Stop. Restore the backup. Recheck the URL and the vendor's page from a second network |
| `grep` finds nothing in `sources.list.d` | The repository is defined in `/etc/apt/sources.list` or a `.sources` file | `grep -r hashicorp /etc/apt/` |

## 9. Automation equivalent

For several servers, the same three actions (back up, download, verify) can be scripted. The fingerprint check must stay in:

```bash
#!/usr/bin/env bash
set -euo pipefail

KEYRING=/usr/share/keyrings/hashicorp-archive-keyring.gpg
EXPECTED="D55C0D1AC78A8D8126CB631CFC9CA96ACA026560"
TMP=$(mktemp)

wget -qO- https://apt.releases.hashicorp.com/gpg | gpg --dearmor > "$TMP"
ACTUAL=$(gpg --no-default-keyring --keyring "$TMP" --fingerprint --with-colons | awk -F: '/^fpr/ {print $10; exit}')

if [ "$ACTUAL" != "$EXPECTED" ]; then
    echo "Fingerprint mismatch: got $ACTUAL" >&2
    rm -f "$TMP"
    exit 1
fi

sudo cp "$KEYRING" "$KEYRING.bak"
sudo install -m 644 "$TMP" "$KEYRING"
rm -f "$TMP"
sudo apt-get update
```

This script has not been run in the lab. Test it on one server before using it on others. The expected fingerprint is written into the script on purpose: when the vendor rotates the key again, the script fails safely until a person confirms the new fingerprint and updates it.

## 10. Finding past fixes by keyword

Before solving a problem again, search for where it was documented. Use distinctive strings from the error, such as an error code or key ID, not general words.

| Where to search | Command |
| --- | --- |
| Files in one repository | `git grep -n -i "NO_PUBKEY"` |
| With surrounding lines | `git grep -n -i -C 3 "hashicorp"` |
| All repositories under one folder | `grep -r -i -n -l "NO_PUBKEY" ~/dev/projects --include="*.md"` |
| Commit messages | `git log --all -i --grep="hashicorp" --oneline` |
| Text added or removed in any commit | `git log --all -S "NO_PUBKEY" --oneline` |
| GitHub, across all your repositories | `NO_PUBKEY owner:<your-username>` in the GitHub search box |

For this issue the search returned nothing, because the vendor's key change was new. This document now exists so that the next search finds it. The **Search keywords** line at the top holds the exact error strings for that purpose.

## 11. Design notes

- **Per-repository keys.** The `signed-by` option ties one key file to one repository, so a vendor's key can only vouch for that vendor's packages. This replaced the older `apt-key` method, which trusted every key for every repository.
- **Keys are fetched once.** Most install instructions download the vendor key a single time. Any vendor key rotation will cause this error on every server set up before the rotation. Configuration management that compares the key's fingerprint on each run would catch it automatically.
- **Do not bypass the check.** Options such as `[trusted=yes]` or `--allow-unauthenticated` silence the error by turning off signature verification for that repository. They should not be used as a fix.
- **Fleet impact.** Any other lab server with the HashiCorp repository, added before 2026-09-09, will show the same error and needs the same fix.

## 12. Open items

- [x] Confirm `sudo apt-get update` runs clean on the Ubuntu DevOps server (Step 5). Confirmed 2026-11-16.
- [ ] Remove the `.bak` file on the Ubuntu DevOps server.
- [ ] Check the other Ubuntu servers in the lab for the same error.
- [ ] The server reported `*** System restart required ***` at login. Schedule a reboot.

## Sources

- HashiCorp key file: https://apt.releases.hashicorp.com/gpg
- HashiCorp published fingerprints: https://www.hashicorp.com/trust/security
- Reports of the 2026-09-09 rotation and the same error: https://github.com/roycohen2013/workstation/pull/19 , https://github.com/ben-everly/ansible/pull/25 , https://github.com/tunacinsoy/omawsl/pull/55

## Document history

| Date | Change | By |
| --- | --- | --- |
| 2026-11-16 | First issue | Techworks |
