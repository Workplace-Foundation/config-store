---
name: update-windows-os-versions
description: Monthly check of Windows 10/11 minimum OS versions in compliance/compliance_os_versions.json (the Windows10 and Windows11 sections, plus the Windows-Notification-Template-Text.CU-month text), against Microsoft's regular Patch Tuesday releases and the Windows Insider blog. Use when the user says things like "check windows updates", "update windows os versions", or asks to walk through/guide through the Windows OS version update.
---

# Update Windows OS versions (monthly check)

This is a recurring, interactive process. The user approves every change one
channel at a time — this skill is a checklist, not a script to run
unattended. Never bulk-apply changes without walking through them as
described below. Covers Windows only - see the separate
`update-apple-os-versions` skill for macOS/iOS/iPadOS.

## 0. Repo hygiene (do this first, every time)

```
git status --short                    # must be clean before switching
git checkout main
git fetch origin --prune
git merge --ff-only origin/main
git branch -v                         # any local branch fully merged into main ([gone]) -> git branch -d it
git checkout -b YYYY-MM-DD-update-windows-os-versions
```

Use today's actual date for the branch name. Only delete local branches that
`git merge-base --is-ancestor <branch> main` confirms are merged — never
force. Keep this as its own branch/PR, separate from any Apple update done
the same day.

## 1. The one-month-lag rule

**Standing rule, confirmed by the user**: in month N, apply month N-1's
regular (second-Tuesday) Patch Tuesday level. E.g. in November, apply
October's level; in December, November's; and so on. Don't ask which level
to use each time - this is settled. (The only thing to ask about is the
*year* boundary - confirm if genuinely ambiguous, e.g. January applying
December of the *previous* year.)

Compute the target month's exact Patch Tuesday date (second Tuesday) before
searching for builds, so you don't accidentally pick up an out-of-band or a
later month's release.

## 2. Read current state

Read `compliance/compliance_os_versions.json`'s `Windows10` and `Windows11`
sections, and `Windows-Notification-Template-Text.CU-month`.

## 3. Source of information

- Regular Patch Tuesday builds: search for `"<Month Day, Year>" KB Windows
  1<0|1> <channel> build` style queries, and cross-check against Microsoft's
  own `support.microsoft.com/.../servicing/os/...` KB articles when a
  result links one - they're authoritative over third-party trackers, which
  sometimes swap 22H2/23H2 build numbers (22621.x is the 22H2 base,
  22631.x is 23H2 - watch for this specific mix-up).
- Beta/Experimental Insider channel flights: fetch
  `https://blogs.windows.com/windows-insider/` for the most recent post
  list, then fetch the most recent "Announcing new builds for ..." post
  directly and ask it which channel(s) and build number(s) it covers - the
  channel isn't always obvious from the title alone.

## 4. Walk through each channel, one at a time

For every tracked key (Windows10: 22H2; Windows11: 22H2, 23H2, 24H2, 25H2,
any Beta/Experimental entries, 26H2, 26H1, Future):

1. Find that channel's regular build for the target month (see step 1).
2. If unchanged: state "no change" and move on (still show it, don't skip
   silently).
3. If changed: present old → new with the KB number and release date, cite
   sources, then **wait for explicit approval** before editing the file.
4. Watch for these gotchas:
   - **Frozen/EoS channels**: `Windows11.22H2` is frozen (End of Servicing)
     - never propose a bump for it without the user raising it first.
   - **A channel reaching its own EoS mid-cycle** (e.g. 24H2 Home/Pro EoS on
     2026-10-13): flag it explicitly and ask whether to freeze the key (like
     22H2) or keep bumping it (e.g. because the fleet is Enterprise/Education,
     which is serviced longer) - don't assume either way.
   - **Hotpatch-affected channels (24H2/25H2/26H2)**: see step 5, don't skip
     the baseline-month check.
   - **Insider channel branch changes**: an Insider channel (Beta,
     Experimental) can have its underlying build-number branch retired and
     moved to a new one mid-cycle (this happened in November 2026: Beta
     channel moved from branch 26220 to branch 26320 via an enablement
     package). When this happens:
     - Explain the mechanism plainly before proposing anything: an
       enablement package is the same lightweight flip used for annual
       feature updates, delivered through ordinary Windows Update: any
       device still on the old branch upgrades to the new one on its own,
       it is not stuck (caveat: Windows Insider flights can still be
       staged/gradual, so a small delay for some devices isn't impossible -
       there's no per-device rollout visibility like Apple's GDMF
       `SupportedDevices` to check this against).
     - Ask the user whether to retire the old key or keep it frozen as a
       safety net (don't default to either). If kept frozen, name the
       retirement reason and the branch-move date in its comment, and add a
       `TODO: remove once confirmed empty` note.
     - Name the new key to match the existing convention (full branch
       number, e.g. `26H2-Beta26320`, not a shortened form) and place it in
       the same relative position the old key held, unless told otherwise.
5. Apply each approved change immediately with Edit before moving to the
   next channel — don't batch silently.

## 5. Hotpatch baseline-month check (24H2, 25H2, 26H2 only)

These three channels can run Windows Hotpatch, which has its own quarterly
cadence: one "baseline" month per quarter (restart required, same build for
hotpatch-enrolled and non-enrolled devices) followed by two "hotpatch" months
(no restart, hotpatch-enrolled devices get a *lower* build number than the
regular monthly release for the same content).

Before proposing a build for 24H2/25H2/26H2, check Microsoft's official
Hotpatch release notes page (e.g.
`support.microsoft.com/en-us/servicing/os/hotpatch/windows-11/2025/release-notes-for-hotpatch-on-windows-11-enterprise-version-25h2`)
for the target month:

- **Baseline month**: use the regular build - hotpatch and non-hotpatch
  devices converge on it, no divergence to handle.
- **Hotpatch (non-baseline) month**: **use the lower hotpatch-track build**,
  not the higher regular/restart build. The higher regular build is a
  stricter minimum than hotpatch-enrolled devices can reach that month,
  which would mark fully-patched hotpatch devices non-compliant. (Established
  rule, confirmed by the user.)

Always state in the file's `_comment` which kind of month it was and which
build was chosen, so the next review doesn't have to re-derive it.

## 6. Summarize, then commit/push/PR

- Show a full before/after table (all tracked channels, not just the
  changed ones) before committing.
- Commit message: list every change with old → new and the KB/source; call
  out explicitly any EoS decision, branch-retirement, or hotpatch
  baseline/non-baseline determination made this cycle, and flag anything
  relevant for *next* cycle (e.g. "next month is a hotpatch month, use the
  lower build then").
- `git push -u origin <branch>`, then `gh pr create` with a body mirroring
  the commit message. Assignee defaults to `@me` unless told otherwise.
  Reviewers only if asked.
- If there's genuinely nothing to change this cycle, say so — no empty
  commits. Delete the branch if it ends up empty.
