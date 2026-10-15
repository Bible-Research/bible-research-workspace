---
name: implement-plan
description: Implement a plan markdown file end-to-end with
  subagents — run parallel frontend/backend implementation agents
  (one draft PR per repo), then review the PRs with fresh reviewer
  agents and iterate fix rounds until reviewers report no issues.
  Invoke when the user asks to implement/execute a plan file
  (e.g. *-plan.md) using agents.
---

# Implement a Plan File with Subagents

Orchestrate a multi-agent implementation of a plan file (e.g.
`note-order-move-plan.md`). This workspace has two repos:
`bible_research/` (Django backend, GitHub
`Bible-Research/bible_research`) and `reactive-bible/` (React
frontend, GitHub `Bible-Research/reactive-bible`).

Subagents are stateless: front-load every prompt with the plan
path, the target repo dir, the branch name, the conventions files
to read, and exactly what to return. Agents share the working
tree — never let two agents write in the same repo concurrently.

Track the phases with `todo_write`.

## Phase 0 — Read the plan

Read the plan file yourself. Extract: a short `<slug>` for branch
names, which repos it touches (`backend`, `frontend`, or `both`),
and the API contract both sides must agree on (endpoint paths,
request/response shapes).

## Phase 1 — Implementation (parallel general agents)

Launch one `subagent_general` **per repo the plan touches**, in
background so they run in parallel. Each prompt must include:

    Implement the <backend|frontend> portion of <plan-file>
    (absolute path) in the <repo-dir>/ repo.

    1. cd <repo-dir>; git switch main && git pull --ff-only;
       git switch -c feat/<slug>
    2. Read <conventions-file> first
       (bible_research/AGENTS.md or reactive-bible/AGENTS.md).
       Every line <= 79 chars in every file type.
       Backend: NEVER run makemigrations/migrate — if a
       migration is needed, write the model change and report
       the command instead.
    3. Implement ONLY this repo's part of the plan. The other
       repo is being changed in parallel — implement against
       the contract the plan specifies, do not wait for it.
    4. Add/update tests per the plan. Verify:
       backend:  source venv/bin/activate &&
                 python -m pytest <changed-app tests> -q
       frontend: npm run lint && npm run build && npm test
    5. Commit only files you edited (never git add -A, never
       commit plan/scratch md). Format:
       `Type: Capitalized msg`.
       Push: git push -u origin feat/<slug>
    6. Open a DRAFT PR with --base main following
       .devin/workflows/create-pr.md at the workspace root —
       write the body to pr-body.md, use --body-file, rm it
       after. Body must include a UI-verification section and
       reference the other repo's PR if the plan has both.
    7. gh pr checks <N> --watch; fix failures until green, then
       gh pr ready <N>.
    Return: PR URL, files changed, test results, anything you
    could not do.

## Phase 2 — Review (fresh reviewer agents)

When every implementation PR exists, launch ONE NEW
`subagent_general` reviewer PER PR, in background. New agents —
never reuse the implementers. Prompt:

    Review PR <url> in <repo-dir>. Do NOT modify any files —
    review only. The PR implements <plan-file> (absolute path);
    read the plan section for this repo.
    1. gh pr diff <N> and read surrounding code for context.
    2. Check: correctness vs the plan, edge cases, the
       error/error_code contract, cross-repo contract match
       (field names, endpoints — diff the sibling PR <url>
       too if it exists), tests actually cover the change,
       79-char limit, no plan/scratch files in the diff, no
       migrations added.
    3. Optionally run the repo's scoped tests.
    Return: numbered findings, each with severity
    (blocker|should-fix|nit) and file:line. Say "NO ISSUES"
    explicitly if clean.

## Phase 3 — Fix loop

If any reviewer reported blocker/should-fix findings:

1. Launch one `subagent_general` fixer per affected PR (the PR
   branch is still checked out in that repo — verify with
   `git -C <repo-dir> branch --show-current`, else switch to it).
   Prompt: the PR URL, the numbered findings verbatim, "fix,
   commit (`Fix: ...`), push, re-watch `gh pr checks`, return
   what changed". Nits may be batched into one fixer or skipped
   — say which.
2. After fixes land, launch NEW reviewer agents (fresh round,
   same review prompt + "previous findings: <list> — verify the
   fixes").
3. Repeat until a review round reports NO ISSUES on every PR,
   or after 3 rounds. If issues persist past 3 rounds, stop and
   report the remaining findings to the user.

## Guardrails

- Background agents cannot request new permissions. If one is
  blocked by a denied tool (e.g. `git push`, `gh`), resume it in
  the foreground so the user can approve.
- Fixers for the same PR run sequentially — one at a time.
- If a subagent reports the plan is unimplementable as written,
  stop and ask the user rather than improvising.

## Final report

PR URLs, review rounds needed, findings fixed per round, and any
outstanding items (e.g. a migration command the user must run).
