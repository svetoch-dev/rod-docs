# Shared rules for Rod agents

## Work order

1. Read the working repository's root `AGENTS.md` and the shared context at its `rod-docs` tag. Inspect the branch and working tree. Use the [map](map.md) and [registry](repositories.md) to find the owner of the change.
2. Resolve dependency refs from the working repository. Read the affected directories' instructions, any applicable skill, and the relevant code at those refs. If required instructions are unavailable, pause work that depends on them. Flag conflicts between instructions and code at the selected ref.
3. Establish the goal, affected paths, approach, checks, and material risks. Confirm the plan when the scope is open, and ask before expanding an agreed scope. Ground decisions in repository evidence and check affected interfaces and consumers.
4. Make the smallest change that meets the goal. Reuse existing modules and patterns, preserve user edits and compatible interfaces, and keep product configuration in the consuming repository.
5. Review the diff and run the repository's relevant checks and configured hooks. Report the result, checks actually run, and remaining gaps. Report significant confirmed issues outside the task separately with evidence and impact.

## Complex decisions

- Handle routine work directly. For complex or architectural decisions, use two assistants when available and authorized. Choose complementary specialties covering implementation and failure modes. Give both the same question, sources, and constraints, and ask each for a recommendation, evidence, and the main risk.
- Have the assistants cross-check each other's findings once. The lead verifies disputed facts against repository evidence and brings unresolved decisions to the user. Assign read-only work by default and separate file sets for parallel edits. Use the smallest suitable non-flagship model and reasoning effort within the user's limits.

## Limits

- Keep real organization names, consumer paths, and cloud identifiers out of reusable defaults and examples. Use neutral fixtures. Keep credentials, secret values, raw state, plans, and sensitive outputs or logs out of AI context. For infrastructure dependencies, inspect source declarations and schemas.
- Inspect a helper's effects before running it. Prepare changes and local checks. Apply and deploy through the established CI path. Get explicit authorization for operations that access remote state, run infrastructure plans, bootstrap or modify cloud resources, or change a live cluster. Local instructions cannot grant that authority.
- Stage only task changes, inspect the staged diff, and group related edits in one local commit. Show the diff and checks before requesting push authorization. Push, remote changes, and PR creation each require explicit authorization for the action. Get approval before rewriting history or discarding user edits. For an authorized PR, summarize the problem, solution, affected modules, impact, and checks.
