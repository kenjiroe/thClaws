# Fork Workflow (kenjiroe/thClaws)

> This file is local to this fork's operating process — it is not part
> of any upstream contribution and should not be included in PRs sent
> to `thClaws/thClaws`.

## 1) Core-impacting changes — issue first, always

If a change touches any of the following, **open a GitHub issue on
`thClaws/thClaws` before writing the implementation**, the same way
Phase 5 (`PHASE5-TOOL-CALL-AUDIT.md`, issue #203) was handled. This is
non-negotiable — it's what keeps this fork mergeable and keeps us from
building on assumptions upstream later invalidates.

**Treated as "core" (issue required):**
- `crates/core/src/**` — anything under core, especially
  `hooks.rs`, `permissions.rs`, `confine.rs`, `mcp.rs`, `gateway*`
- Policy schema changes (anything that changes what a `policy.json`
  block can contain, or how it's validated/signed)
- Anything affecting the `pre_tool_use` gate contract or
  `ApprovalSink`/`ApprovalDecision` semantics
- Anything affecting `confine.rs` mode selection or fail-open/closed
  behavior
- Public CLI flags/config keys in `crates/core` that ship in the OSS
  build (not GUI-only cosmetic changes)

**Not required (safe to just build in the fork):**
- Fork-only tooling, CI, docs, or Spec Kitty scaffolding
- Experiments kept on a throwaway branch, never merged to this fork's
  `main`
- GUI-only changes that don't touch `crates/core`

**Process when core-impacting:**
1. Write the design (a `PHASE<N>-<NAME>.md` doc at repo root, same
   style as `PHASE5-TOOL-CALL-AUDIT.md`) before code.
2. `gh issue create --repo thClaws/thClaws` linking the design doc on
   this fork's branch.
3. Wait for maintainer feedback (do not block other fork work on this
   — see §3).
4. Implement in a `feature/<name>` branch in this fork regardless of
   whether maintainer has replied yet.
5. If maintainer aligns → open PR to `thClaws/thClaws:main`. If they
   don't respond or decline → keep the feature in this fork's `main`
   and flag it in this file's "§4 Fork-only deltas" list so it's
   tracked in one place.

## 2) Upstream sync schedule

Run this **weekly**, or immediately before starting any new
core-impacting feature branch (to minimize conflicts):

```bash
cd ~/Github/thClaws
git checkout main
git fetch upstream
git log --oneline main..upstream/main   # see what's new before merging
git merge upstream/main                  # or: git rebase upstream/main
git push origin main
```

Prefer `merge` over `rebase` on `main` once this fork has its own
merged commits on `main` (§4 deltas) — rebasing rewrites history that
other feature branches may already be based on. Use `rebase` freely on
short-lived `feature/*` branches that haven't been pushed for review
yet.

**After syncing `main`, rebase active feature branches on top:**

```bash
git checkout feature/phase5-tool-call-audit
git rebase main
git push --force-with-lease origin feature/phase5-tool-call-audit
```

**If a rebase/merge conflicts inside `crates/core/**`:** stop and
re-read the upstream commit that introduced the conflict — it may mean
upstream shipped something that makes the fork's approach obsolete
(this already happened once this session: `confine.rs`/`permissions.rs`
turned out to already cover ground we assumed was missing based on the
README alone). Don't just resolve textually; re-verify the fork's
design doc still makes sense against the new upstream code.

## 3) Fork-only deltas (tracked)

Features developed in this fork that have not (yet, or ever) been
merged upstream. Keep this list current so "what's different from
upstream" is answerable in one place instead of a full `git diff`.

| Feature | Branch | Upstream issue | Status |
|---|---|---|---|
| Phase 5: client-side tool-call audit log | `feature/phase5-tool-call-audit` | [thClaws/thClaws#203](https://github.com/thClaws/thClaws/issues/203) | Proposed — awaiting maintainer feedback |

## 4) Quick reference

```bash
# weekly sync
git checkout main && git fetch upstream && git merge upstream/main && git push origin main

# rebase a feature branch after sync
git checkout feature/<name> && git rebase main && git push --force-with-lease origin feature/<name>

# open an alignment issue before core work
gh issue create --repo thClaws/thClaws --title "..." --body-file <design-doc>.md
```
