# Process change — Pre-retire content diff is mandatory

**Date:** 2026-05-05
**Author:** Plumb (Claude / Mac M4 Desktop)
**Triggering event:** AMP-118 vault consolidation — the naive "retire redundant vaults" plan would have silently destroyed 831 unique voice notes.

## The rule

**Before retiring any data store, vault, repo clone, or similar content-bearing path, run a content diff against the canonical. Set difference > 0 = NOT retire-eligible without a drain step first.**

This applies regardless of how confident the retire decision feels. "Looks like a duplicate" is not enough — it must be *verified* as a duplicate by content comparison.

## Why this rule

AMP-118 inventory found six paths called "vault" or vault-adjacent. Surface inspection suggested most were redundant clones of the canonical `Amplified-Partners/vault`. A naive retire would have deleted all of them. **Content diff revealed:**

- `~/Manual Library/Projects/vault` — actually IS the canonical clone, in sync with `Amplified-Partners/vault`. 0 ahead / 0 behind. Was on the retire list. Would have been a major loss.
- `~/Manual Library/real-vault/_inbox-voice/` — 831 unique markdown files (voice-note transcripts dated Apr 25 onward), none of which existed in `Projects/vault/_inbox-voice/`. Set intersection: zero. Would have been a silent ~3-week data loss.

Both were caught only by file-level diff. Filename inspection or directory-name inspection would not have caught either.

## What actually happened (the underlying pattern)

`real-vault` is git-cloned from `https://github.com/ewan-dot/amplified-vault.git` — Ewan's **personal** GitHub account.

`Projects/vault` is git-cloned from `https://github.com/Amplified-Partners/vault.git` — the **organisation** account (canonical).

A launchd job `~/Library/LaunchAgents/com.amplified.vault-sync.plist` runs `~/Manual Library/real-vault/scripts/vault-sync.py` and pushes new content to `ewan-dot/amplified-vault`. It has been running continuously and capturing voice notes since at least 2026-02-25 (most recent: 2026-05-01 nightly sync).

Around 2026-02-25, the canonical vault was migrated/forked to `Amplified-Partners/vault` and `Projects/vault` was set up as the working copy. **The auto-sync launchd job was never migrated to the new canonical.** Two parallel repos drifted apart for ~10 weeks.

This is the **third instance** of the same pattern across the estate:
1. `amplified-crm` (Mac local) → `ewan-dot/amplified-crm` vs canonical `Amplified-Partners/crm` ([AMP-82](https://linear.app/amplifiedpartners/issue/AMP-82))
2. `gatekeeper-agent` (Mac local) → `ewan-dot/gatekeeper-agent` (no org canonical) ([AMP-80](https://linear.app/amplifiedpartners/issue/AMP-80))
3. `real-vault` (Mac local) → `ewan-dot/amplified-vault` vs canonical `Amplified-Partners/vault` (this finding)
4. `token-proxy` (Mac local) → `ewan-dot/anthropic-token-proxy` vs canonical `Amplified-Partners/anthropic-token-proxy` (audit finding from earlier in session, not yet ticketed)

**Pattern:** when the canonical repo migrates from a personal GitHub account to the org, automation that points at the personal account is left behind. Two parallel repos diverge silently. Eventually someone tries to consolidate and almost loses content.

## The process change (mandatory steps for any future consolidation)

For any retire / wipe / consolidation work touching content-bearing paths:

1. **Inventory** — list every candidate path with size, file count, file-type breakdown.
2. **Identify automation** — for each candidate, scan `~/Library/LaunchAgents`, `~/Library/LaunchDaemons`, crontab, systemd timers, git hooks, supervisord configs, etc. for processes that WRITE to the path. Document the writer for each.
3. **Identify remote** — `git config --get remote.origin.url` per path. Flag any pointing to non-canonical accounts (typically `ewan-dot/*` or other personal forks).
4. **Content diff** — for each candidate, compute the set difference of files vs. the canonical. Sample some content. Confirm zero unique signal-bearing content.
5. **Drain unique content** — if set difference > 0, copy the unique content into the canonical, push, verify ingestion (e.g., Graphiti entity count delta, file appears in canonical clone).
6. **Reroute automation** — point any writers at the canonical, not the soon-to-be-retired path. Verify one full cycle of the writer hits the canonical.
7. **Stage retire** — move retiring paths to `~/_consolidated_<date>/retired/` (Hazel pack staging convention). NOT delete — stage.
8. **Wait for confidence** — wait ≥ 24 hours so any missed writer / process has a chance to surface (it'll fail to write and complain).
9. **Then retire.**

Skipping any of steps 4–6 is what almost lost 831 voice notes on AMP-118.

## Codification

This finding lives in `mirror/findings/` for Plumb's record. Cross-references:
- AMP-118 (vault consolidation — Phase 2 now blocked on this drain step)
- AMP-82 (CRM identity mismatch — same pattern)
- AMP-80 (gatekeeper-agent PAT exposure — same pattern with credentials added)

Worth proposing to Sam for inclusion in the Low-Hanging Fruit / Compound Engineering loop as a recurring "Sense" pattern: parallel-repo drift detection.

## Attribution
*Plumb (Claude on Mac M4 Desktop, Anthropic) — 2026-05-05*
*Triggered by Ewan's "well saved... that needs to go in as a process changed. It needs analysed as to what happened."*
