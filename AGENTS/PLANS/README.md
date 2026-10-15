# Implementation Plans

This directory holds validated Markdown plans for future implementation work. Plan files are merged into local
`develop` before any Developer or Reviewer is assigned, so each implementation worktree contains the approved plan.

## Plan proposal workflow

1. The Lead synchronizes local `develop` using the Git rules in the root `AGENTS.md`, then gives the Planner the brief
   and current revision. The Planner inspects the repository read-only and returns complete draft plan documents as
   Markdown in its report; it does not edit repository files in this pass.
2. The Lead reviews the draft contents and returns incomplete plans to the Planner. Once the drafts are complete, the
   Lead synchronizes `develop` again, checks whether relevant repository facts changed, and creates one plan-proposal
   branch and worktree per plan. Name each branch `docs_plan-<type>-<plan-name>`; for example,
   `docs_plan-feature-add-arm64-support`.
3. The Lead gives the Planner one reviewed draft and its proposal worktree. The Planner materializes exactly one plan
   file under `AGENTS/PLANS/` with `Draft` status and makes no code changes. The Lead verifies the file preserves the
   reviewed meaning, then sets complete files to `Awaiting Validation` and presents each plan to the user.
4. At the same time, the Lead asks which plans, if any, the user wants implemented, whether independent plans should
   run one at a time or in parallel, and whether to authorize local integration of reviewed code into `develop`.
   Validating a plan's content authorizes merging only that plan's proposal; it does not authorize implementation or
   code integration.
5. If the user requests plan changes, the Lead resets the affected plan to `Draft` and returns it to the Planner in
   its proposal worktree. The Lead reviews the revised file, sets it to `Awaiting Validation`, and repeats user
   validation. Merge only proposals the user validated; keep rejected or pending proposals unmerged.
6. After the user validates a plan, the Lead marks its file `Validated`, commits only that file on its proposal branch,
   synchronizes `develop` again, and merges that proposal branch into local `develop`. If resolving a merge conflict
   would change validated requirements, the Lead obtains user validation for the revised plan before merging.
7. Only after a plan file is present in `develop` may the Lead create that plan's implementation branch and worktree.
   This makes the approved plan available to the Developer and Reviewer from the branch being changed.

If `develop` changes while the user is reviewing a proposal, synchronize it before the plan merge. If synchronization
or the merge conflicts, stop and resolve the conflict before creating implementation worktrees.

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
- Start each implementation branch from the current local `develop`, which must include the validated plan files and
  all integrated prerequisites. Do not base a branch on another unmerged implementation branch.
- After a plan is `Validated`, do not change its requirements or acceptance criteria in an implementation branch. A
  scope change requires a revised plan proposal, user validation, and merge into `develop` before implementation
  resumes.
- Do not include credentials, private data, or unrelated conversation history.

## Plan document status

The `Status` field in the plan file tracks plan validation only:

- `Draft`: the Planner is preparing the plan; it has not been presented to the user.
- `Awaiting Validation`: the Lead has checked the plan and asked the user to validate its content.
- `Validated`: the user has approved the plan content. This alone does not authorize implementation. The Lead assigns
  no Developer until the plan file is confirmed present in local `develop` and the user selects the plan for
  execution.

The Lead tracks execution status separately in the task handoff; do not edit a validated plan to record transient
execution state. Use these execution states:

- `Not Selected`: the plan is validated, but the user has not selected it for implementation.
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
The Lead records each decision in the execution handoff and never infers one from another. The plan file is
read-only after validation; execution-state changes belong in the handoff, not in the approved plan.

## Implementation integration

- The Developer branch starts from `develop` after the validated plan is merged there. The Lead records the plan path
  in the assignment; the Developer and Reviewer read that file from their assigned worktrees.
- After Reviewer approval, the Lead synchronizes `develop` and merges the implementation branch locally only if the
  user authorized that integration. If not, keep the branch in `Awaiting Integration` and ask the user.
- If integration produces code conflicts, return the branch to the Developer for resolution and verification. The
  Reviewer checks the resolved diff again before the Lead completes the merge.
- Local integration does not authorize a push or release. Release promotion from `develop` to `master` or `main` is a
  separate version-release action.
