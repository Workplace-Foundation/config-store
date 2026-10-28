---
name: update-apple-os-versions
description: Weekly check of Apple (iOS/iPadOS and macOS) minimum OS versions in compliance/compliance_os_versions.json and configuration-policies/macos_update_target_os_versions.json, against Apple's live GDMF feed. Use when the user says things like "new week, new check for apple updates", "update apple os versions", "check apple updates", or asks to walk through/guide through the Apple OS version update.
---

# Update Apple OS versions (weekly check)

This is a recurring, interactive process. The user approves every change one
major at a time — this skill is a checklist, not a script to run unattended.
Never bulk-apply changes without walking through them as described below.

## 0. Repo hygiene (do this first, every time)

```
git status --short                    # must be clean before switching
git checkout main
git fetch origin --prune
git merge --ff-only origin/main
git branch -v                         # any local branch fully merged into main ([gone]) -> git branch -d it
git checkout -b YYYY-MM-DD-update-apple-os-versions
```

Use today's actual date for the branch name. Only delete local branches that
`git merge-base --is-ancestor <branch> main` confirms are merged — never force.

## 1. Fetch Apple's GDMF feed

GDMF (`gdmf.apple.com/v2/pmv`) is Apple's own authoritative device-management
feed — the same source MDM vendors use. On this network it fails plain TLS
validation (`UntrustedRoot` / self-signed cert in chain), so it needs:

```powershell
Invoke-WebRequest -Uri "https://gdmf.apple.com/v2/pmv" -SkipCertificateCheck -OutFile <path>
```

- If the fetch fails for any reason — the auto-mode permission classifier
  denies it (TLS/Auth Weaken), a network/timeout error, a non-200 response,
  etc. — **always show the user an explicit warning saying the GDMF download
  was not successful** (don't let it pass silently), then fall back to
  WebSearch for each major instead (it has worked fine in past weeks). Do
  this every time it happens, not just once.
- Do **not** try to route around a classifier denial via curl/Bash with `-k`
  or similar — that's the same bypass under a different name.
- If the user asks to save the raw file somewhere (e.g. Downloads), download
  it directly with `Invoke-WebRequest -OutFile`, not a re-serialized/pretty-
  printed copy — they asked for the *original source* before and a
  PowerShell-reformatted copy isn't it. Default to Downloads, not any
  project-specific reports folder, unless told otherwise.

To extract the relevant entries (adjust the major-version list if new ones
get added), filter `SupportedDevices` so you don't pick up watchOS/tvOS/
HomePod entries that share the same `iOS`/`macOS` buckets in the feed:

```powershell
$j = Get-Content <path> | ConvertFrom-Json
# iOS/iPadOS: keep only iPhone*/iPad* devices
foreach ($e in $j.PublicAssetSets.iOS) {
    if ($e.SupportedDevices | Where-Object { $_ -like "iPhone*" -or $_ -like "iPad*" }) { ... }
}
# macOS: ProductVersion starting with the tracked majors (14/15/26/27 etc.)
```

Always also check each `SupportedDevices` list when a brand-new
`ProductVersion` shows up for a major you don't expect yet (see step 3,
device-exclusive releases).

## 2. Read current state

Read both files:
- `compliance/compliance_os_versions.json` — sections `iOSiPadOS` and `macOS`
- `configuration-policies/macos_update_target_os_versions.json` — `default_ring`

## 3. Walk through each major, one at a time

For every tracked key (iOSiPadOS: 15, 16, 17, 18, 26, 27, Future; macOS:
Sonoma, Sequoia, Tahoe/whatever is current "latest", GoldenGate/whatever is
current "newest", Future):

1. Compare current `Build` to GDMF's build for that major.
2. If unchanged: state "no change" and move on (still show it, don't skip
   silently — the user wants the full walkthrough).
3. If changed: WebSearch to corroborate — get the release date, what it
   fixes (CVE? severity? reliability-only?), and cite sources. Present the
   old → new value with the "why", then **wait for explicit approval**
   before editing the file.
4. Watch for these gotchas before proposing a bump:
   - **Device-exclusive releases**: a new `ProductVersion` in GDMF whose
     `SupportedDevices` lists only one specific model (e.g. a new phone/
     foldable) is not a general fleet release. Flag it, don't propose it as
     the new baseline, and keep tracking the prior general build until a
     general release (e.g. "iOS 27.1" exclusive to one device vs. "iOS 27.2"
     for everyone).
   - **Channel status changes**: if a macOS entry you're tracking as a
     preview/Release-Preview-style build has actually gone GA, say so
     explicitly and ask whether to repurpose the key rather than silently
     relabeling it.
   - **Device-coverage regressions (the mirror case)**: don't just take "the
     highest version number" for a major at face value — compare the
     candidate build's `SupportedDevices` list against the build currently
     in the file (or, if GDMF still lists the current build too, against
     that entry directly). If any device model covered by the current build
     is *missing* from the candidate's `SupportedDevices`, some already-
     supported devices can't install it yet. This has actually happened:
     not all 26.x devices could update to the then-newest point release:
     only the *next* release after that was available fleet-wide. In that
     situation, do not bump compliance to the newer-but-narrower build -
     say so explicitly and hold at the current build until a release comes
     out that covers every device the current build does.
   - **"Future" entries**: never move Future except to keep it above the
     highest now-tracked major (i.e. only when a new major gets added).
5. `Future` placeholders aside, apply each approved change immediately with
   Edit before moving to the next major — don't batch silently.

## 4. macOS "latest" gets deferred treatment — this is the key rule

The macOS major that is currently newest (named entry just below `Future`,
e.g. `Tahoe`) is handled differently from every other entry, because of
`configuration-policies/macos_update_target_os_versions.json`'s
`default_ring`:

- `default_ring` is the forced-update **rollout target** — it leads with the
  newest patch and carries a `target_time` (ask the user for the date each
  time; it's historically ~1 week out, at `23:00:00`).
- The **compliance floor** for that same macOS major must **not** move to
  the new build until `default_ring`'s `target_time` has already passed.
  Devices haven't received the update yet before that date — bumping
  compliance early would mark fully-compliant devices as non-compliant.
- So each week:
  1. If GDMF shows a newer build for macOS latest, bump `default_ring.Build`
     to it and set a new `target_time` (ask the user for the date).
  2. Check whether a *previous* `default_ring.target_time` has now passed
     with compliance still lagging behind it — if so, that's the week to
     catch compliance up to that build.
  3. Never let compliance for macOS latest get ahead of a `default_ring`
     target that hasn't passed yet. If this was done by mistake earlier in
     the session, revert it and say so plainly.

## 5. Summarize, then commit/push/PR

- Show a full before/after table (all tracked majors, not just the changed
  ones) before committing.
- Commit message: list every change with old → new and the reason; note any
  macOS-latest deferral explicitly (what's held back and why, and when to
  revisit).
- `git push -u origin <branch>`, then `gh pr create` with a body mirroring
  the commit message. Assignee defaults to `@me` unless told otherwise.
  Reviewers only if asked — resolve first names to GitHub logins via
  `gh api repos/<org>/<repo>/collaborators --paginate` and confirm with
  `gh api users/<login> --jq '.name'` before assuming a match.
- If there's genuinely nothing to change this week, say so — no empty
  commits. Delete the branch if it ends up empty
  (`git log main..<branch> --oneline` should show nothing first).
