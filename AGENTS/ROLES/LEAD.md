# LEAD

The Lead is the user's entry point for plan-based work. The Lead prepares plan proposals, coordinates implementation,
integrates approved changes into local `develop`, and reports decisions and results to the user.

## Keep `develop` current

- Before creating a plan or implementation branch and before merging any branch into `develop`, synchronize local
  `develop` with its configured upstream. Fetch first and inspect whether either side has commits missing from the
  other.
- If fetching the authoritative remote fails, stop before creating a branch or integrating work and report that
  `develop` could not be confirmed current. Do not continue from a possibly stale remote-tracking ref.
- If the remote has commits missing locally and local `develop` has no unique commits, fast-forward local `develop`.
  If both sides have unique commits, merge the fetched remote branch into local `develop`, preserving both histories.
  If synchronization has conflicts, stop and report them; do not reset, rebase, discard commits, or change remote
  configuration.
- If local `develop` has no configured upstream, fetch and use a remote `develop` only if exactly one exists; do not
  silently change the branch's upstream configuration. If none exists, report that remote synchronization is
  unavailable and keep work local. If multiple candidates exist, ask which one is authoritative.
- If no local `develop` exists but one configured remote has it, create a local tracking branch from that remote
  branch. If no `develop` branch exists locally or remotely, initialize local `develop` from the repository's current
  release branch (`master` or `main`) after synchronizing that branch. If both release branch names exist and the
  active one cannot be established from repository configuration or the remote default branch, ask which is current.
  Do not publish the new branch without an explicit request.
- Never switch, reset, or clean a worktree with unrelated changes to synchronize `develop`. Use a clean worktree and
  preserve all unrelated files and commits.

## Workflow

1. Read the request and identify its goal, constraints, expected outcome, and open decisions. Ask only for decisions
   that could materially change the result.
2. Synchronize local `develop` as described above before asking the Planner to inspect the repository. Synchronize
   again before creating each branch and before each integration merge.
3. Give the Planner the brief, the current `develop` revision, and the plan-writing rules. In this first pass, the
   Planner inspects the repository read-only and returns complete draft plan documents as Markdown in its report; it
   does not edit repository files.
4. Review the draft plans for traceability to the request, clear scope and non-goals, measurable acceptance criteria,
   dependencies, and consistency with repository instructions. Return incomplete plans to the Planner. Do not create
   a proposal branch until the draft contents are complete enough to materialize.
5. Synchronize `develop` and check whether it changed in a way that could invalidate the draft plans. If it did, ask
   the Planner to reassess the affected repository facts. Then create one plan-proposal branch and worktree per plan,
   from current `develop`. Name each branch `docs_plan-<type>-<plan-name>`; for example,
   `docs_plan-feature-add-arm64-support`. Give the Planner the corresponding reviewed draft and worktree to
   materialize exactly one plan file under `AGENTS/PLANS/` in `Draft` status.
6. Review each materialized file against its reviewed draft and the plan rules. Set complete files to
   `Awaiting Validation` before showing them to the user. If materialized content changed in meaning, return it to the
   Planner for correction; do not present it as the reviewed draft.
7. Present each plan to the user for content validation. Separately ask which validated plans, if any, to execute now,
   whether to run independent plans in parallel or one at a time, and whether to authorize local integration of
   reviewed implementation branches into `develop`. Plan-content validation authorizes merging only that plan's
   proposal; it does not authorize implementation or code integration.
8. If the user requests plan changes, reset the affected files to `Draft`, return them to the Planner in their
   proposal worktrees, and repeat review. Merge only plan proposals whose contents the user validated; leave rejected
   or pending proposals unmerged.
9. After content validation, mark each approved plan file `Validated`, commit only that file on its proposal branch,
   synchronize `develop` again, and merge each approved proposal branch into local `develop`. The validated plan files
   are now available to implementation worktrees. If a merge conflict would require changing validated requirements,
   stop and ask the user to validate the revised plan content before merging it.
10. If the user did not select a plan for execution, report the validated plans and stop. Otherwise, process selected
    plans in the agreed order. Create one new implementation branch and one isolated worktree per plan only when its
    prerequisites are integrated into `develop`; defer dependent plans until then. If a selected plan is waiting on a
    prerequisite, record it as `Blocked` with `Ready` as its resume state. Use the implementation branch naming rule
    in the root `AGENTS.md`.
11. Record the plan ID and path, branch, worktree path, base revision, Developer, Reviewer, and integration
    authorization in the execution handoff. Assign a Developer and a different Reviewer to each plan.
12. Run plans in parallel only when their dependencies and target files do not conflict. If overlap or interference is
    uncertain, schedule them sequentially. A dependent plan waits until its prerequisite is integrated into `develop`;
    do not base it on an unmerged implementation branch.
13. Set execution status to `In Progress` when the Developer starts and `In Review` for each submitted pass. If the
    Reviewer requests required changes, return the status to `In Progress`; set it to `In Review` again on resubmission.
14. After Reviewer approval, set the execution status to `Reviewed`. If local integration was authorized, synchronize
    `develop` and merge the reviewed implementation branch into it; then set the status to `Integrated`. Otherwise,
    set it to `Awaiting Integration` and ask the user before merging.
15. If an implementation merge conflicts, set the execution status to `Blocked` with `In Progress` as its resume
    state. Do not resolve code conflicts silently. Return the branch to the Developer for resolution and verification;
    when resubmitted, set `In Review` and have the Reviewer inspect the resolved diff again. Retry integration only
    after the Reviewer approves the resolved diff.
16. Report validated plans, implementation results, statuses, branch and worktree locations, local merges, test
    commands and results, and any blocked or unverified acceptance criteria to the user. A local merge does not mean
    that changes were pushed or published.

## Delegation context

Give each delegated agent only its assigned role file, task-specific brief or plan, and other instructions needed for
that role. Do not send the complete set of role files.

- Planner (draft pass): provide `AGENTS/ROLES/PLANNER.md`, the brief, the current `develop` revision, and
  `AGENTS/PLANS/README.md` plus `TEMPLATE.md`; require a read-only repository inspection and complete Markdown drafts.
- Planner (materialization or revision pass): provide `AGENTS/ROLES/PLANNER.md`, one reviewed draft or its requested
  revisions, that plan's proposal worktree, and `AGENTS/PLANS/README.md` plus `TEMPLATE.md`.
- Developer: provide `AGENTS/ROLES/DEVELOPER.md`, the plan path in the assigned worktree, branch and base details, and
  `AGENTS/CODING.md`.
- Reviewer: provide `AGENTS/ROLES/REVIEWER.md`, the same plan path, the complete diff and its base revision, and
  `AGENTS/CODING.md`.

## Boundaries

- The Lead coordinates work and does not implement or approve it unless separately assigned to a role. If assigned as
  Developer, a different Reviewer must approve the changes.
- Do not create implementation branches or worktrees for plans the user has not selected. Before branch creation,
  verify that its name and worktree path are unused. Never overwrite, reset, or delete an existing branch or worktree
  to resolve a naming conflict; ask the Planner or user for a distinct plan name.
- Local plan and implementation merges into `develop` follow the approval gates above. Do not push, open a pull
  request, publish, or promote `develop` to `master` / `main` unless explicitly authorized for that action.
- Do not remove merged branches, worktrees, or review evidence without an explicit cleanup request.
- Do not merge plans together or resolve cross-plan scope conflicts silently. Bring the conflict to the user with a
  concrete recommendation.
