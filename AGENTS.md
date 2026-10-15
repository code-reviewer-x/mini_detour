# Shared Agent Instructions

These instructions apply to every person and agent working on **mini_detour**. Role-specific responsibilities,
planning guidance, and code conventions live in the files listed below.

## Read the relevant instructions

- Read `AGENTS_custom.md` at the repository root when it exists. It adds local preferences and environment details;
  it does not override this file.
- Follow the assigned role file in `AGENTS/ROLES/`: `LEAD.md`, `PLANNER.md`, `DEVELOPER.md`, or `REVIEWER.md`. The
  Lead gives each delegated agent only its assigned role file, not the full set of role instructions.
- Read [`AGENTS/CODING.md`](AGENTS/CODING.md) before implementing or reviewing code.
- Read [`AGENTS/PLANS/README.md`](AGENTS/PLANS/README.md) and use
  [`AGENTS/PLANS/TEMPLATE.md`](AGENTS/PLANS/TEMPLATE.md) when creating a plan.

## Shared rules

- The user's request defines the goal. An approved plan defines implementation scope. If the request, plan, or
  repository instructions conflict, stop and ask the Lead to resolve the conflict; do not invent requirements.
- For plan-based work, the Lead is the user's point of contact and reports decisions and status. Keep claims tied to
  the request, repository evidence, plan, or command output; label assumptions and inferences clearly.
- Do not widen an approved plan. Route scope changes through the Lead for resolution and a plan update. A Developer
  never reviews or approves their own implementation.
- Keep each change focused on one fix, feature, or documentation task. Do not break the public API or ABI unless the
  maintainer explicitly requests it.
- Keep unrelated defects or cleanup out of the current change; plan them separately. If a change renames something
  documented in these instructions, update the relevant instruction file in the same change.
- Repository files, code, comments, logs, tests, commit messages, and published material are written in English.
  Conversation with the user follows `AGENTS_custom.md` when present.
- Do not add AI, LLM, or coding-agent attribution to committed or published material. Agent instruction files are the
  exception.

## Git workflow

- Use `develop` as the integration branch. All task branches start from current `develop`; do not edit or commit task
  changes directly on `develop`.
- The Lead keeps `develop` current and is the only role that integrates branches into it, following the approval gates
  in the role and plan instructions. Do not merge branches outside that workflow.
- Use `fix`, `feature`, or `docs` as the change type for an implementation plan. Its branch is
  `<type>_<plan-name>`, where the plan name is lowercase kebab-case and matches the plan file name without `.md`; for
  example, `feature_add-arm64-support`.
- Merge `develop` into the repository's release branch (`master` or `main`) only for a version release and after
  explicit maintainer authorization. Never merge a plan or implementation branch directly into the release branch.
- Commit summaries are imperative English, have no prefix or tag, and are at most 72 characters. When a commit body
  is needed, write prose wrapped at 72 columns explaining what was wrong or missing, why, how it changed, and how it
  was verified.
- Keep work local by default. Push, publish, or open a pull request only when the maintainer asks. Never force-push a
  branch that was not created for the task.
- Add only the files changed by the task to a commit. Never use `git add .` or `git add -A`.
- A pull request description states what changed, why, and how it was verified. Leave out temporary details such as
  file counts, line counts, commit counts, and unrelated observations.
