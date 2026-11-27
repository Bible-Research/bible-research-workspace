---
name: fix-staging-bug
description: >
  Fix a bug the user spotted on the reactive-bible staging
  preview (Vercel) — pasted dev-console errors, a "why am I
  seeing X on staging" report, or misbehavior reproduced on
  the staging deploy. Use when the user describes or pastes a
  staging frontend symptom and asks to fix it. Investigates,
  assigns the fix to a background agent in its own worktree,
  opens a `--base staging` PR, runs the reviewer loop until
  clean, merges to staging on green, and leaves a `--base
  main` promotion PR open. Invoke as /fix-staging-bug.
---

# Fix a staging bug (reactive-bible)

The pattern for bugs found on the staging preview
(`bible-research-git-staging-<team>.vercel.app`). Target repo
is `reactive-bible/` (GitHub `Bible-Research/reactive-bible`,
remote `origin`); all work happens in worktrees — never the
main checkout (`reactive-bible/AGENTS.md`).

Environment: prefix every exec call with
`export PATH="/opt/homebrew/bin:$PATH"` — `gh`, `node`,
`npm` are not on the default PATH.

## 1. Triage before assigning

Explain the symptom first — a wall of console errors is not
always a defect:

- Grep the source for the exact log strings (`"Failed to
  prefetch audio"`, `"Prefetched next chapter"`) and the
  minified stack's owning module to locate the code path.
- Distinguish real defects from expected noise: an expected
  condition logged at error level (e.g. a guaranteed-404
  prefetch for a book outside the fileset's coverage) is a
  defect worth fixing — but the fix is gating/demoting, not
  hiding a real failure. Interactive-path errors that drive
  correct UI state (e.g. `BookNotInFilesetError` showing the
  "not in this fileset" hint) stay untouched.
- If the root cause is a backend contract
  (`error`/`error_code`, fileset routing), say so and work in
  `bible_research` per its AGENTS.md instead — same agent +
  staging-PR flow does not apply (backend is trunk-based).

## 2. Assign the fix to an agent

Spawn a `subagent_general` background agent per bug. Give it
the full context it cannot see: repo path, the triaged root
cause and exact file:line pointers, and this workflow:

- Worktree + branch, off `origin/main` (never `staging`):
  `git -C reactive-bible worktree add worktrees/<slug> \
  -b fix/<slug> origin/main`, then `npm install` inside.
- Failing test reproducing the symptom first, then fix.
  79-char lines everywhere. Follow existing test conventions
  (MSW, `vi.mock('../api')`).
- Verify: `npm run lint` (0/0), `npm run build`, scoped
  `npx vitest run` on touched areas.
- Update `DEVELOPER_GUIDE.md` if behavior changed.
- Stage only own files; commit `Fix: Capitalized message`;
  push `-u origin`.
- PR `--base staging` (ready, never draft), body via a temp
  file + `gh pr create --body-file` (delete temp after).
  Body must include a `## UI verification / QC` section —
  concrete steps on the affected page/route.
- `gh pr checks <N> --watch`; fix until green, then report —
  the agent does NOT merge or remove its worktree.

If one agent's branch already exists for the same area (a
staging fix for a regression its own PR introduced), restore
that branch at the merged staging PR's `headRefOid`
(`git push origin <sha>:refs/heads/<branch>`) and have the
fix land on it — that keeps the open `--base main` promotion
PR's head updated automatically.

## 3. Reviewer loop (orchestrator)

For each green PR, spawn a fresh `subagent_explore` reviewer
on the diff (`gh pr diff <N>` — note: `subagent_explore` has
no exec tool, so paste the diff into its prompt or point it
at a worktree containing the branch). Fix every useful
finding — small fixes directly in a fresh worktree on the
branch, larger ones via a fixer agent — push, and repeat
with a new reviewer until a round reports nothing meaningful.

## 4. Merge to staging

Once checks are green and the reviewer loop is clean:

    HEAD=$(gh pr view <N> --json headRefOid --jq .headRefOid)
    gh api -X PUT \
      "repos/Bible-Research/reactive-bible/pulls/<N>/merge" \
      -f merge_method=squash -f sha="$HEAD"

The `-f sha=` is the atomic head check — never merge a
stale SHA.

## 5. Open the `--base main` promotion PR — leave it open

GitHub auto-deletes the head branch on merge; re-push the
exact merged head first (pushing the SHA, not a branch —
a deleted ref makes a plain `git push` a silent no-op):

    git push origin "$HEAD":refs/heads/<branch>

Then `gh pr create --base main --head <branch>` reusing the
staging PR's title and body verbatim (keep `Closes #<N>`
links) and append the marker line:

    _Promoted from `staging` (PR #<N>)_

Leave the main PR OPEN — the user merges it.

Caveats:

- A staging PR whose conflicts forced `git merge
  origin/staging` into the branch makes its head carry the
  whole queue — the main promo is then valid only
  merge-last (its residual diff after earlier promos equals
  the feature). Add a merge-order note to that PR's body;
  per AGENTS.md such branches are really train-only
  (`Promote to main` workflow).
- The `staging-first` gate on the main PR requires the head
  SHA to have merged into staging — a head that absorbed
  follow-up commits after the staging merge fails the gate
  until those commits are staged too (or promote the merged
  SHA, not the branch tip).
- A red check on the main PR that already passed on the same
  SHA for staging may be flake — re-run the job
  (`gh run rerun <run-id> --failed`). If it recurs, fix the
  test on the ORIGINATING feature branch (so its own open
  main promo picks it up), merge that staging fix first,
  then re-run.

## 6. Finish

- Remove the local worktree and branch
  (`git worktree remove --force worktrees/<slug>`,
  `git branch -D <type>/<slug>`); `git worktree list` should
  show only the main checkout.
- Report: what the errors meant, root cause, staging PR
  merged, main PR URL left open, reviewer-loop outcome,
  anything logged as expected-behavior/no-fix.
