---
name: arch-maintenance
description: Cautious, audit-first rulebook for maintaining an Arch Linux system with pacman and the paru AUR helper. Use this skill whenever the user wants to audit, update, upgrade, or clean an Arch system; check for available package updates; review AUR packages or PKGBUILDs; handle .pacnew/.pacsave files; remove orphan packages; clean the pacman or paru cache; plan a full-system upgrade; or verify the system after an upgrade. Trigger it even when the user only says things like "check my Arch box", "is it safe to update", "what needs upgrading", "clean up pacman", or "review this AUR package", because every one of those touches the no-partial-upgrade and approval-gated rules below. Offline Arch Wiki and local man pages are the source of truth ("the wiki wins" on conflicts); never answer Arch maintenance facts from model memory when local verification is available.
---

# Arch maintenance

A conservative doctrine for maintaining an Arch Linux system. This skill is a rulebook, not an automatic updater. The default posture is **audit-first and approval-gated**: gather facts, report, and propose; never mutate the system without explicit approval from the user.

The system has a `paru` AUR helper installed. Repo packages come from `sudo pacman`; AUR packages come from `paru`.

## Core operating principles

1. **Audit before action.** Always start read-only. Collect state, classify findings, propose a plan. Mutation is a separate, approved step.
2. **The wiki wins.** Verify every Arch fact against the offline Arch Wiki or local man pages before stating it. Do not answer from model memory when local verification is available, because Arch changes fast and memory drifts. Cite what you consulted.
3. **Nothing is destroyed to fix something.** Recovery over destruction. Roll back, ignore, or pin; never resolve a problem by discarding state.
4. **Approval gates every change.** Any command that mutates the system (upgrade, remove, clean, overwrite config) is proposed with its purpose and risk, then waits for an explicit yes.

**The approval boundary, concretely.** Read-only audit commands (everything in Phase 1, plus `checkupdates`, `paru -Qua`, `paru -Ps`, `paru -Gp/-Gc`, `informant list --unread`, `checkrebuild`, `pacdiff -o`, `wiki-search`, diffs, `find`) are run *directly*: they change nothing, so do not stall the audit asking permission for each one; just run them and report. The gate applies only to commands marked **changes system: yes** (upgrade, `-Rns`, `paccache`/`-Scc`, overwriting a config with a `.pacnew`). Tag every command you propose as read-only or changes-system so the user knows exactly what needs a yes. This is what makes the skill useful unsupervised: it gathers the full picture on its own, then stops at the one line that matters.

## Non-negotiable rules

These are not style preferences; breaking them is how Arch systems break.

- **Full upgrades only. No partial upgrades.** A partial upgrade (syncing the package databases but only upgrading some packages) produces mismatched shared libraries and is the single most common way to break Arch.
- **Never run `pacman -Sy <package>`.** That refreshes the databases without a full upgrade, which is exactly a partial upgrade. If a package is wanted, the path is a full upgrade first.
- **Use full-upgrade flows only:** `sudo pacman -Syu`, `paru` (a bare `paru` does a full `-Syu` of repo + AUR), or `paru -Syu`.
- **`pacman -Syuw` (download-only) carries the same risk as `-Sy`:** it syncs the databases without installing anything. The safe pre-download is `checkupdates -d`, which downloads pending updates to the cache without touching the live databases.
- **Never use `--overwrite` or `-Rdd`.** `--overwrite` bypasses file-conflict checks and is only justified when an Arch news post explicitly instructs it; `pacman -Rdd` skips dependency checks and can remove a critical dependency. Both are what a stuck transaction tempts you to reach for; neither is a fix.
- **An interrupted `-Syu` must be completed.** If `pacman -Syu` fails partway, the `-Sy` database sync has already succeeded, so the system is already in the partial-upgrade state. Resolve the error and finish the upgrade before any other package operation.
- **Check Arch news before any upgrade.** Offline docs do not carry live manual-intervention notices, so https://archlinux.org/news/ must be checked before recommending or running an upgrade. A flagged manual intervention can require steps before `-Syu` or the upgrade will break the system. This system enforces the check with **informant** (AUR): its pacman hook (`00-informant.hook`) aborts every pacman/paru transaction while unread Arch news exists. When a transaction is blocked, read the news (`informant list --unread`, then `informant read <item>`), act on any manual intervention, and only then let the marked-read state unblock the run. Never clear the gate blindly with `informant read --all`; marking news read without reading it defeats the one mechanism that stops a known-breaking upgrade. Caution: `informant check` is NOT purely read-only; with exactly one unread item it prints and marks it read, so the audit uses `informant list --unread` instead.
- **Treat the AUR as unofficial and untrusted until reviewed.** AUR PKGBUILDs are user-submitted and run with build privileges; read them before installing.
- **Never use `--skipreview`.** Reviewing the PKGBUILD and diff is the entire safety mechanism for the AUR.
- **Avoid blind `--noconfirm` for AUR builds.** Confirmation prompts are a deliberate checkpoint.
- **Prefer official repo packages over AUR packages** when both provide what is needed; repo packages are signed and maintained.
- **Review `.pacnew` and `.pacsave` files**; never blindly delete or blindly overwrite live configs with them.
- **Cleanup is conservative and approval-gated.** Keep recent cache; review orphans before removal.
- **Never delete user data, dotfiles, configs, or snapshots.** Cleanup touches package caches and confirmed orphans only.

## Documentation priority

Consult sources in this order. Higher sources win on conflict.

1. **Offline Arch Wiki via `wiki-search`** (primary).
2. **Offline Arch Wiki HTML** under `/usr/share/doc/arch-wiki/html/en/` (e.g. `Pacman.html`, `Arch_User_Repository.html`); `rg` that directory to search the full text.
3. **Local man pages**: `man pacman`, `man makepkg`, `man paru`, `man paccache`, `man pacdiff`, `man checkupdates`.
4. **Live official Arch Wiki** (wiki.archlinux.org) only when local docs are missing, stale, or the topic likely changed since the snapshot.
5. **Current Arch Linux news** (https://archlinux.org/news/) before any upgrade recommendation; this is mandatory for upgrades, not optional.
6. **Upstream paru documentation** if paru-specific behavior is unclear.
7. **Other sources** only if explicitly needed.

### Using the offline Arch Wiki

The offline wiki is a dated snapshot from the `arch-wiki-docs` package. Report its version when wiki facts matter:

```bash
pacman -Q arch-wiki-docs
```

Two-step search:

```bash
wiki-search <terms>     # prints matching pages numbered 0-9 (title + 8-digit page ID)
wiki-search <n>         # opens result <n> from the immediately preceding search
wiki-search <page-id>   # a partial or full page ID may open a page directly
wiki-search --all       # lists every available title
```

Search is case-insensitive regex / term-list based. For full-text search of the HTML:

```bash
rg -l "<pattern>" /usr/share/doc/arch-wiki/html/en/
```

Because the snapshot is dated, if a question hinges on recent behavior, note the snapshot date and fall back to the live wiki.

## Phased workflow

Keep these phases separate. Do not slide from reporting into mutating. Each mutating phase needs its own approval.

1. **Audit** (read-only). Run the audit command set, classify findings.
2. **Upgrade planning.** Check Arch news, list available updates, flag high-risk categories, propose the upgrade.
3. **Upgrade execution** (only if approved). Full upgrade flow only.
4. **`.pacnew` / `.pacsave` review.** Find them, diff them, merge deliberately.
5. **Cleanup** (only if approved). Conservative cache trim and reviewed orphan removal.
6. **Reboot recommendation.** Recommend a reboot when the kernel, firmware, systemd, or graphics stack changed.
7. **Post-upgrade verification.** Re-check failed units and recent errors.

## Phase 1: Audit (read-only)

Run these to gather state. None of them change the system.

```bash
hostnamectl
uname -a
cat /etc/os-release

pacman -Q arch-wiki-docs

checkupdates 2>/dev/null || true
paru -Qua
paru -Ps
informant list --unread

pacman -Qm
pacman -Qqe
pacman -Qqet
pacman -Qtdq
checkrebuild

systemctl --failed
journalctl -p 3 -xb --no-pager

df -h
du -sh /var/cache/pacman/pkg 2>/dev/null
du -sh ~/.cache/paru 2>/dev/null
stat -c '%y' /etc/pacman.d/mirrorlist

pacdiff -o
sudo find /etc -name "*.pacnew" -o -name "*.pacsave"

ls /usr/lib/modules
uname -r
```

What each group tells you:
- `hostnamectl` / `uname -a` / `os-release`: identity and kernel baseline.
- `pacman -Q arch-wiki-docs`: offline wiki snapshot freshness.
- `checkupdates`: repo updates **without** touching the live database (safe; it uses a temporary database). `paru -Qua`: AUR updates available; `-git`/VCS packages only show updates when devel checking is on (`Devel` in `/etc/paru.conf`, or `--devel`), and the devel check does a network git lookup per package.
- `paru -Ps`: paru's own health summary (orphans, out-of-date AUR packages, packages that no longer exist on the AUR; needs network). `informant list --unread`: unread Arch news items; the informant hook blocks every pacman/paru transaction while any exist.
- `pacman -Qm`: foreign packages, meaning AUR installs plus anything dropped from the official repos (dropped packages stop receiving updates and deserve a call-out). `-Qqe`: explicitly installed. `-Qqet`: explicitly installed and not required by anything (cleanup candidates to review). `-Qtdq`: true orphans (unrequired dependencies).
- `checkrebuild` (from rebuild-detector): foreign packages linked against libraries that have since moved; these need a rebuild, not a reinstall of the same binary.
- `systemctl --failed` / `journalctl -p 3 -xb`: failed units and priority-3 (error) messages this boot.
- `df -h` / `du -sh ...`: disk pressure and cache sizes.
- `stat ... mirrorlist`: mirrorlist age; mirror quality drifts, and a stale mirror can serve old databases, which mimics partial-upgrade symptoms. Refreshing it (see the wiki `Mirrors` page) changes the system, so it is proposed, not run.
- `pacdiff -o`: pending config merges, found via the pacman database's backup entries (read-only). The `find` fallback catches strays the database scan misses; without sudo it prints permission-denied noise for root-only dirs.
- `ls /usr/lib/modules` vs `uname -r`: installed kernel module trees vs the running kernel (mismatch hints a reboot is overdue after a kernel upgrade).

**`checkupdates` availability:** it ships with `pacman-contrib`. Before relying on it, verify it exists (e.g. `command -v checkupdates`). If missing, report that `pacman-contrib` may be needed and proceed without it; do **not** install it without approval. The same pattern applies to `pacdiff` (also `pacman-contrib`), `checkrebuild` (`rebuild-detector`), and `informant` (AUR): probe with `command -v`, report if missing, never install anything without approval.

## Phase 2: Upgrade planning

1. **Check Arch news first** (https://archlinux.org/news/), and `informant list --unread` locally. Surface any manual-intervention notice and its required steps before proposing the upgrade. This cannot be skipped because offline docs never contain these notices. The informant hook will hard-block the upgrade anyway while news is unread, so plan the read-and-act step (`informant read <item>`) into the proposal rather than hitting the block mid-run.
2. List what would change (`checkupdates`, `paru -Qua`).
3. Flag high-risk categories present in the update set (see below).
4. Propose the full-upgrade command with purpose, risk, and a clear "changes system: yes".

### High-risk update categories

When any of these appear in the pending updates, call them out explicitly, recommend reading the relevant news/wiki entry, and recommend a reboot afterward where applicable:

- **kernel** (`linux`, `linux-lts`, etc.) — needs reboot; keep `linux-lts` as a fallback.
- **NVIDIA** drivers — must match the kernel; reboot needed.
- **bootloader** (GRUB / systemd-boot) — a bad config can prevent boot; regenerate config deliberately.
- **systemd** — init and core services; reboot recommended.
- **pacman** itself — may ship `.pacnew` for its own config; re-read before further operations.
- **glibc** — core C library; partial upgrades around it are especially dangerous.
- **openssl** — ABI changes can break many packages at once.
- **mesa** — graphics stack.
- **KDE / Plasma** — desktop environment; large coordinated update, reboot or re-login.
- **firmware** (`linux-firmware`, microcode) — reboot needed to take effect.

## Phase 3: Upgrade execution (approval required)

Only after the user approves and after Arch news is clear:

```bash
sudo pacman -Syu        # repo full upgrade
# or
paru                    # full upgrade of repo + AUR (-Syu)
# or
paru -Syu
```

Never `pacman -Sy` anything. If installing a new package, do it as part of (or after) a full upgrade, never against a partially-synced database.

If the upgrade breaks something, recover without destroying state: roll the offending package back with `pacman -U /var/cache/pacman/pkg/<old-version>` (if that version is no longer cached, the Arch Linux Archive at https://archive.archlinux.org/ keeps every released package), then hold it with `IgnorePkg` until upstream ships a fix. `IgnorePkg` is a short-term shield, not a resting state: holding a package back is itself a mild partial upgrade, and locally built AUR packages need rebuilds when a dependency gets a soname bump. Remove the entry as soon as the fix lands.

## paru and AUR workflow

The AUR is untrusted until reviewed. Before installing or upgrading an AUR package, read what it will actually do:

```bash
paru -Gp package-name    # print the PKGBUILD (inspect build/install logic)
paru -Gc package-name    # print the package's AUR page comments (users flag breakage and malware there); -Gcc for all pages
paru -G package-name     # clone the package build files locally for full inspection (git log / git diff inside the clone)
```

Read the PKGBUILD for anything surprising: network fetches from odd hosts, `sudo`/privilege use, destructive commands, obfuscation. The diff since the last reviewed version appears in paru's install-time review step; that step is exactly what `--skipreview` would bypass. Never run `paru`/`makepkg` as root or with `sudo`. Never pass `--skipreview`. Avoid `--noconfirm` for AUR builds. Prefer a repo equivalent if one exists.

## Phase 4: `.pacnew` / `.pacsave` review

`.pacnew` appears when a package ships a new default config but you have local edits; `.pacsave` appears when a removed package's config is preserved. `pacdiff` (pacman-contrib) is the tool for both finding and merging them:

```bash
pacdiff -o                     # read-only: list pending .pacnew/.pacsave/.pacorig files (pacman db backup entries)
pacdiff -s -3                  # interactive merge via sudoedit; 3-way diff against the cached base package when available
DIFFPROG=<editor> pacdiff -s   # override the default diff program (vim -d)
sudo find /etc -name "*.pacnew" -o -name "*.pacsave"   # thorough fallback; catches strays the db scan misses
```

For each one, keep local customizations and pull in the upstream changes that matter. Do not delete a `.pacnew` without confirming its changes are either applied or genuinely unwanted. Only `pacdiff -o` runs freely; the merge itself overwrites a live config, so it is a review task, presented for approval, not an automatic overwrite.

## Phase 5: Cleanup (approval required)

Cleanup is conservative by default. Review first, then act only on approval.

### Orphan packages

Dry review (read-only):

```bash
pacman -Qtdq
```

Only after the user approves the specific list:

```bash
sudo pacman -Rns $(pacman -Qtdq)
```

Inspect the list before removing; a package showing as an orphan may still be wanted. If it is wanted, the fix is `sudo pacman -D --asexplicit <pkg>` (marks it explicitly installed so it stops surfacing as an orphan; a database-only change, but it changes the system, so propose it). Never expand this into removing explicit packages or user-needed tools.

### Cache cleanup

Conservative trims (keep recent versions so rollback stays possible):

```bash
sudo paccache -r       # keep the last 3 versions of each package
sudo paccache -ruk0    # remove cached versions of uninstalled packages only
```

Avoid defaulting to the nuclear option:

```bash
sudo pacman -Scc       # wipes the entire cache; destroys rollback ability — avoid unless the user explicitly insists and understands the tradeoff
```

Keeping a few cached versions is what makes `pacman -U /var/cache/pacman/pkg/<old-version>` rollback possible.

For the AUR build cache, use paru's own cleaner rather than a blunt `rm -rf ~/.cache/paru`:

```bash
paru -Sc       # clean cached AUR packages and untracked build files (keeps VCS sources); -Scd to delete entirely
```

`paru -Sc` knows what is safe to drop; a hand-rolled `rm -rf` of the cache directory does not and is easy to point at the wrong path.

### Cleanup safety boundary

Cleanup is narrow on purpose: it touches **package caches and confirmed orphans, and nothing else**. Stay inside that boundary because "freeing space" is exactly when an over-eager cleanup deletes something load-bearing.

- **Never hand-delete files that a package owns.** If something under `/usr`, `/etc`, or a module tree looks like dead weight, the owning package is removed with `pacman -R`, never with `rm`. The package manager tracks what is safe to drop; raw `rm` does not.
- **Kernel module trees (`/usr/lib/modules/<version>`) are never cleanup targets.** A stale tree means a kernel package is still installed (or was half-removed); resolve it through pacman, not `rm -rf`. In particular `linux-lts` is the deliberate GRUB fallback kernel on this system — its module tree is not clutter, and proposing to delete it as "cleanup" is a real way to lose your recovery kernel. A running-kernel-vs-modules mismatch is a *reboot* signal (see Phase 6), not a delete signal.
- **Never delete user data, dotfiles, configuration files, or system snapshots** under any cleanup operation.

When a cleanup candidate is anything other than a package-cache tarball or a confirmed orphan, stop and treat it as a question for the user, not an action.

## Phase 6: Reboot recommendation

Recommend a reboot when the upgrade touched the kernel, kernel modules, firmware/microcode, systemd, or the graphics stack (NVIDIA/mesa). The `ls /usr/lib/modules` vs `uname -r` check from the audit shows when the running kernel no longer matches installed modules, which is a strong reboot signal.

## Phase 7: Post-upgrade verification

After an upgrade, confirm the system is healthy:

```bash
systemctl --failed
journalctl -p 3 -xb --no-pager
checkrebuild
```

Report new failed units or new priority-3 errors that were not present in the pre-upgrade audit, and tie them to what changed. Any package `checkrebuild` newly lists is linked against a library the upgrade moved and needs a rebuild (`paru -S <pkg>` rebuilds an AUR package); that rebuild changes the system, so propose it.

## Report format

Always summarize an audit or plan using this structure. Classify every finding into exactly one bucket, and attach purpose/risk/changes-system to every proposed command so the user can approve with full context.

```text
Arch Maintenance Report

Documentation checked
- <wiki pages / man pages / news consulted, with snapshot version where relevant>

Critical
- <things that will break the system or block boot/upgrade>

Should fix
- <real problems that are not yet critical>

Optional cleanup
- <orphans, cache, cosmetic items>

No action
- <checked and healthy>

Proposed action plan
1. ...
2. ...
3. ...

Commands requiring approval
- command:
  purpose:
  risk:
  changes system: yes/no
```
