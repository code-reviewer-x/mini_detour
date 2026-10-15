# Plan Lifecycle

This directory contains the plan template, lifecycle rules, and three directories for plan files. A plan's directory
is its canonical lifecycle state; plan files do not contain a `Status` field. The Lead moves plan files between
directories with Git so every transition is recorded.

## Plan directories

- `PENDING_PLANS/`: new drafts and plans awaiting user validation. No Developer starts work from a plan in this
  directory.
- `VALIDATED_PLANS/`: plans whose content the user approved. They may be waiting for implementation, in
  implementation or review, or awaiting authorized local integration into `develop`.
- `FINISHED_PLANS/`: plans whose implementation the Developer reports complete, whose changes the Reviewer approves,
  and whose completion the Lead confirms after local integration into `develop`.

Moving a plan to `FINISHED_PLANS/` records completion in the local repository; it does not mean the branch was pushed
or a release was published. If local integration is not authorized or cannot proceed, keep the plan in
`VALIDATED_PLANS/` and track `Awaiting Integration` in the execution handoff.

## Plan proposal workflow

1. The Lead synchronizes local `develop` using the Git rules in the root `AGENTS.md`, then gives the Planner the brief
   and current revision. The Planner inspects the repository read-only and returns complete draft plan documents as
   Markdown in its report; it does not edit repository files in this pass.
2. The Lead reviews the draft contents and returns incomplete plans to the Planner. Once the drafts are complete, the
   Lead synchronizes `develop` again, checks whether relevant repository facts changed, and creates one plan-proposal
   branch and worktree per plan. Name each branch `docs_plan-<type>-<plan-name>`; for example,
   `docs_plan-feature-add-arm64-support`.
3. The Lead gives the Planner one reviewed draft and its proposal worktree. The Planner creates exactly one plan file
   under `PENDING_PLANS/` using [`TEMPLATE.md`](TEMPLATE.md). The Planner makes no code changes. The Lead checks that
   the file preserves the reviewed meaning and presents it to the user for content validation.
4. Separately ask which plans, if any, the user wants implemented, whether independent plans should run one at a time
   or in parallel, and whether to authorize local integration of reviewed implementation branches into `develop`.
   Content validation authorizes merging only that plan's proposal; it does not authorize implementation or code
   integration.
5. If the user requests changes, keep the plan in `PENDING_PLANS/`, return it to the Planner in its proposal worktree,
   review the revision, and ask the user to validate it again. Do not merge a rejected or unvalidated proposal into
   `develop`; keep its proposal branch unmerged as the review record.
6. After the user validates a plan, move it from `PENDING_PLANS/` to `VALIDATED_PLANS/` with `git mv`. Commit that
   move on the proposal branch, synchronize `develop` again, and merge the proposal branch into local `develop`. The
   approved plan is available to implementation worktrees only after this merge. If resolving a merge conflict would
   change the validated content, obtain user validation for the revised content before merging.
7. Only after the plan file is present in `VALIDATED_PLANS/` on `develop` may the Lead create its implementation
   branch and worktree. The Developer and Reviewer use the plan from that directory.

If `develop` changes while the user is reviewing a proposal, synchronize it before merging the plan. If
synchronization or the merge conflicts, stop and resolve the conflict before creating implementation worktrees.

## Plan files

- Create one file per independently reviewable deliverable, using the name
  `<type>_<plan-name>.md`. `<type>` is `fix` for a defect or broken behavior, `feature` for new behavior, platform
  support, tests, or CI, and `docs` for documentation-only changes. `<plan-name>` is lowercase kebab-case. For
  example: `feature_add-arm64-support.md`.
- The implementation branch uses the same name without `.md`. Each selected plan gets one new implementation branch
  and one dedicated worktree; never share a worktree between plans or reuse a branch that already exists.
- Use [`TEMPLATE.md`](TEMPLATE.md). Replace every placeholder. Write all plan content in English.
- Keep the plan specific to the user request and repository evidence. Do not add speculative requirements or present
  a recommended implementation as if the user requested it.
- A plan must state dependencies, scope, non-goals, measurable acceptance criteria, and how acceptance will be
  verified. Write `None` when a section has no entries; do not leave a section ambiguous or silently omit it.
- Name source files or symbols only when repository inspection supports the reference. Mark uncertain technical
  details as assumptions or questions.
- Split work into separate plans only when each can be reviewed on its own. If one plan depends on another, state the
  dependency by plan ID and explain what must be complete first.
- Start each implementation branch from current local `develop`, which must include the validated plan file and all
  integrated prerequisites. Do not base a branch on another unmerged implementation branch.
- After a plan enters `VALIDATED_PLANS/`, do not change its requirements or acceptance criteria in an implementation
  branch. A scope change requires a revised proposal, user validation, and merge into `develop` before implementation
  resumes. For a revision, create a proposal branch from current `develop`, move the plan from `VALIDATED_PLANS/` to
  `PENDING_PLANS/` in that branch, and revise it there. The copy in `develop` remains unchanged until the revised
  proposal is approved and merged back into `VALIDATED_PLANS/`.
- Do not include credentials, private data, or unrelated conversation history.

## Execution status

The Lead tracks execution status separately in the task handoff; directory location tracks plan lifecycle. Use these
execution states:

- `Not Selected`: the user approved the plan content, but has not selected it for implementation.
- When the user selects a plan, transition from `Not Selected` to `Ready` if all prerequisites are integrated. If a
  prerequisite is not yet integrated, use `Blocked` with `Ready` as the resume state.
- `Ready`: the user selected the plan for implementation and all prerequisites are integrated. Transition to
  `In Progress` when the Developer starts.
- `In Progress`: the Developer is implementing the plan or resolving required review findings. Transition to
  `In Review` every time the Developer submits a pass.
- `In Review`: the assigned Reviewer is checking the submitted pass. If any `Required` finding remains open,
  transition back to `In Progress`; after the Developer submits changes, transition to `In Review` again. Transition
  to `Reviewed` only after the Reviewer explicitly approves with no open `Required` findings. `Optional` findings do
  not block approval.
- `Reviewed`: the Reviewer approved the implementation. If local integration is authorized and can proceed, the Lead
  synchronizes `develop` and merges the branch, then transitions to `Integrated`. Otherwise, transition to
  `Awaiting Integration`.
- `Awaiting Integration`: the Reviewer approved the implementation, but local integration into `develop` is not
  authorized or cannot yet proceed. After authorization and prerequisites are in place, the Lead synchronizes and
  merges into `develop`, then transitions to `Integrated`.
- `Integrated`: the Lead merged the reviewed implementation branch into local `develop`.
- `Blocked`: work in any active state cannot continue without a decision or external change. The Lead records the
  blocker, the state to resume, and the next action. Once resolved, resume at the recorded state; do not treat
  unverified work as reviewed or integrated.

Plan-content validation, implementation authorization, and local integration authorization are separate decisions.
The Lead records each decision in the execution handoff and never infers one from another.

## Completion and archival

- The Developer reports that every plan acceptance criterion is met and provides verification evidence.
- The Reviewer explicitly approves the implementation with no open `Required` findings and confirms the acceptance
  evidence supports the plan.
- The Lead independently checks the report and review, confirms the acceptance criteria are met, and locally
  integrates the reviewed implementation branch into `develop`.
- Only after all three agree and integration succeeds does the Lead create a lifecycle branch and worktree from the
  updated local `develop`, named `docs_finish-<type>-<plan-name>`. Move the plan with `git mv` from
  `VALIDATED_PLANS/` to `FINISHED_PLANS/`, commit only the move on that branch, synchronize `develop`, and merge the
  lifecycle branch into local `develop`. Do not change the plan content during archival. This branch keeps the rule
  that task changes are merged into `develop`, not committed directly on it.
- If implementation changes are required after completion review, leave the plan in `VALIDATED_PLANS/`; return it to
  the Developer and Reviewer cycle. If integration is unauthorized or blocked, leave it in `VALIDATED_PLANS/` and
  track the execution status separately.
