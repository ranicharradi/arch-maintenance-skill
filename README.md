# arch-maintenance

A cautious, audit-first Arch Linux maintenance skill for AI coding agents. Works with Claude Code and any other agent that reads the [Agent Skills](https://agentskills.io) format (a `SKILL.md` with YAML frontmatter).

It is a rulebook, not an automatic updater. The agent gathers system state read-only, reports findings in a fixed format, and proposes every mutation (upgrade, cleanup, config merge) as a separate step that waits for explicit approval. The rules that keep Arch alive are hard-coded: full upgrades only, never `pacman -Sy`, never `--overwrite` or `-Rdd`, AUR packages treated as untrusted until the PKGBUILD is reviewed, recovery over destruction.

## What the agent does with it

- **Read-only audit**: pending repo and AUR updates, unread Arch news, failed systemd units, journal errors, orphans, foreign packages needing rebuilds, pending `.pacnew`/`.pacsave` merges, cache sizes, mirrorlist age, kernel-vs-modules mismatch.
- **Upgrade planning** gated on Arch news, with high-risk categories (kernel, NVIDIA, glibc, systemd, bootloader, ...) called out explicitly.
- **Approval-gated execution**: full-upgrade flows only, `.pacnew` review with `pacdiff`, conservative cache and orphan cleanup, reboot recommendation, post-upgrade verification.

Every proposed command is tagged `changes system: yes/no` so you always know exactly what needs a yes.

## Requirements

- Arch Linux with `pacman` and the [`paru`](https://github.com/Morganamilo/paru) AUR helper
- `pacman-contrib` (provides `checkupdates`, `paccache`, `pacdiff`)
- [`rebuild-detector`](https://github.com/maximbaz/rebuild-detector) (provides `checkrebuild`) for finding AUR packages that need rebuilds after library bumps
- [`informant`](https://github.com/bradford-smith94/informant) (AUR), a pacman hook that blocks transactions while Arch news is unread
- Optional: `arch-wiki-docs` (offline Arch Wiki) plus a `wiki-search` helper script; without them the agent falls back to the live wiki

The skill degrades gracefully: it probes each tool with `command -v`, reports what is missing, and never installs anything without approval.

## Installation

### Claude Code

Personal (available in every project):

```bash
git clone https://github.com/ranicharradi/arch-maintenance-skill ~/.claude/skills/arch-maintenance
```

Project-level (available in one repo):

```bash
git clone https://github.com/ranicharradi/arch-maintenance-skill .claude/skills/arch-maintenance
```

Verify: ask the agent "is it safe to update my arch box?" and it should announce it is using the skill and start a read-only audit.

### Other agents

Any agent supporting the Agent Skills format discovers a skill by its directory containing `SKILL.md`. Clone this repo into wherever your agent looks for skills, keeping the directory name `arch-maintenance`:

```bash
git clone https://github.com/ranicharradi/arch-maintenance-skill <your-skills-dir>/arch-maintenance
```

## Usage

Trigger it with ordinary requests, for example:

- "check my Arch box"
- "is it safe to update?"
- "what needs upgrading?"
- "clean up pacman"
- "review this AUR package before I install it"
- "handle my .pacnew files"

The agent answers with a classified report (Critical / Should fix / Optional cleanup / No action) plus a proposed action plan, and stops before running anything that changes the system.

## Adapting to your setup

Two details assume the machine this skill was written on and are safe to edit in `SKILL.md`:

- **News gate**: the skill expects `informant` as the pacman news hook. If you use something else (for example paru's `NewsOnUpgrade`), swap those references; the "check Arch news before any upgrade" rule itself is not optional.
- **Fallback kernel**: `linux-lts` is named as the deliberate fallback kernel; adjust if you keep a different one.

## Evals

`evals/evals.json` contains behavioral test prompts with expected outcomes (audit stays read-only, AUR review before install, conservative cleanup). Useful as regression checks if you modify the skill.
