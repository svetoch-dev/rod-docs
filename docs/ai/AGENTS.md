# Shared rules for Rod agents

## Work order

Answer generic questions directly. For read-only Rod consultations, use steps 1 and 2 to ground the answer in relevant repository context. For any requested edit, complete steps 1–3 before editing, then follow steps 4 and 5.

1. Read the working repository's root `AGENTS.md` and the shared context at its `rod-docs` tag. Inspect the branch and working tree. Use the [map](map.md) and [registry](repositories.md) to find the owner of the change.
2. Resolve dependency refs from the working repository. Read the affected directories' instructions, any applicable skill, and the relevant code at those refs. If required instructions are unavailable, pause work that depends on them. Flag conflicts between instructions and code at the selected ref.
3. Prefill the [task card](#task-card). Use the [Planner → Executor → Critic workflow](#planner--executor--critic) when its conditions apply. Ground the Planner's plan in repository evidence and check affected interfaces and consumers. Send the completed card and plan together to the user and obtain explicit approval before editing. Ask before expanding an agreed scope.
4. Make the smallest change that meets the goal. Reuse existing modules and patterns, preserve user edits and compatible interfaces, and keep product configuration in the consuming repository.
5. Review the diff and run the repository's relevant checks and configured hooks. Report the result, checks actually run, and remaining gaps. Report significant confirmed issues outside the task separately with evidence and impact.

### Task card

- Goal and purpose
- Deliverable and form
- Scope and constraints
- Requirements and examples where supplied
- Acceptance criteria and checks
- Material open decisions and separate approvals

Fill the card from the user's request and repository evidence. Reference applicable instructions without repeating them. Ask only for missing information or a decision on a conflict that materially affects the card or plan.

## KISS and YAGNI

- Apply these rules to code, configuration, automation, and documentation. Reuse existing components and suitable built-ins. Choose one suitable tool per task and use its built-in integrations where they meet the need.
- Prefer the shortest clear, linear solution. Use descriptive names, including for loop variables. Minimize concepts, functions, and nesting.
- Implement only what the task requires. Skip speculative features, premature optimization, extra wrappers, custom factories, and custom provider abstractions that hide or swap underlying tools or services. Use required integrations directly.
- Keep required checks and add checks for concrete failure modes.

## Planner → Executor → Critic

Use this workflow for complex work, material decisions, or an explicit user request, including a short test. The coordinator dispatches the roles in order, each in a fresh context, accepts the plan and final result, and obtains required approvals from the user. Share only the checkpoint path, goal anchor (or proposed card before approval), and relevant files or refs, plus the accepted plan for the Executor or the candidate diff and check evidence for the Critic.

For a routine edit outside this workflow, the coordinator follows the Planner rules below and requests user approval under Work order.

1. **Planner (read-only).** Read the proposed card or existing anchor and relevant files. **Plan-and-Solve.** Use at least U3 depth by breaking the goal into work packages and each package into concrete steps. Order steps by dependency and give the expected result and check for each step. Include risks and approval needs in the plan. **Tree of Thoughts.** At a material fork, compare two or three viable approaches against the goal and acceptance criteria in the anchor using the available facts. Record the choice, its basis, and when to reconsider it. Explain the goal, deliverable form, proposed approach, and why it fits in a brief user-facing update. The coordinator requests approval of the card and plan on the Planner's behalf.
2. **Executor (scoped edits).** Read the anchor, relevant files, and accepted plan. **ReAct.** For each step, choose and take one permitted action, then compare the observed result with the expected result. Proceed if the result matches, adjust within scope, or return to the coordinator for replanning. Stop before an action that requires approval under the Git or environment rules. Resume that action only after the coordinator confirms explicit user approval for the action and target. Give brief user-facing updates when a work package completes or a material deviation or blocker arises. Return the candidate diff, checks actually run and their results, and any blockers.
3. **Critic (read-only).** Review the candidate diff and check evidence against the anchor and relevant files. Independently recheck the diff and relevant results. Give a brief user-facing verdict of `pass`, `revise`, or `blocked` with evidence against each acceptance criterion and any required corrections. Refer disputes that affect acceptance to the coordinator, who asks the user to decide and pauses dependent work until resolved.

For `revise`, the Executor corrects the work and repeats only affected checks. The coordinator sends the revised diff and affected check evidence to a fresh Critic. Report the first blocker immediately to the coordinator, along with any material uncertainty or approval needs. The coordinator resolves the blocker, replans, obtains the required approval, or leaves the task open.

Accept completion after a Critic `pass` and once the acceptance criteria and approval requirements are met. If fresh contexts are unavailable and independent review is optional, the coordinator may accept after a same-context review against the same criteria. Disclose that review used the same context. If independent review is required, leave the task pending until it occurs.

### Goal anchoring and checkpoints

- Use the approved task card as the goal anchor for changes. For work without a card, record the goal and deliverable, acceptance criteria, scope, constraints, and required approvals in the anchor. Change the anchor only on the user's direction.
- Keep a compact checkpoint with the current checkout, ref, phase, accepted plan, cumulative verified decisions and results, modified files, blockers, and the next action. Include check and review evidence or durable references to it. Update the checkpoint after a verified milestone, handoff, or blocker and before a context reset.
- Keep the anchor and checkpoint in `STATE.md` at the root of the local task workspace, outside version control. Pass its path in handoffs and when pausing. Exclude secrets and raw logs. On resumption, recheck the anchor, current files, and permissions before acting.

## Specialist review for complex decisions

- When a complex or architectural decision needs additional specialist input, use two assistants if available and authorized. Choose complementary specialties covering implementation and failure modes. Give both the same question, sources, and constraints, and ask each for a recommendation, evidence, and the main risk.
- Have the assistants cross-check each other's findings once. The lead verifies disputed facts against repository evidence and brings unresolved decisions to the user. Assign read-only work by default and separate file sets for parallel edits. Use the smallest suitable non-flagship model and reasoning effort within the user's limits.

## Git

- Stage only changes for the task. Inspect the staged diff, run relevant checks, and group related changes in one local commit. Show the user the commit, diff, and check results before asking to push.
- Wait for explicit approval before pushing, creating or changing a PR, merging, or changing remote branches or tags. Approval for one action covers only that action. Flag any deployment a push or merge may trigger before asking for approval.
- Preserve existing work. Get explicit approval before deleting files, branches, or tags, discarding changes, or rewriting history. This includes destructive `git reset`, `git clean`, and `git restore` operations, as well as `git commit --amend`, `git rebase`, and force-push. Identify what the operation will affect.
- For an authorized PR, summarize the problem, solution, affected modules, impact, and checks.

## Permissions and environments

- By default, prepare source changes and run local checks. Inspect the effects of scripts, build targets, and workflows before invoking them. Credentials, tool access, and local instructions do not grant permission to change an environment.
- Never apply, deploy, or change a live environment directly, including development, test, and production. Examples include `terraform apply`, `terraform destroy`, Terraform state writes, `helm install`, `helm upgrade`, `helm uninstall`, Kubernetes and cloud writes, and Argo CD sync or prune. This restriction also applies when a script, bootstrap command, build target, or manually triggered workflow would perform the operation. Infrastructure changes follow the established CI path after user approval.
- Never initiate deletion of resources or data on your own. Before preparing a change that would remove deployed resources, obtain explicit approval for the target and affected environments. Use the established procedure for the approved change.
- Get explicit approval for the action and target before running a Terraform plan or accessing remote state. Keep credentials, secret values, raw state, plans, and sensitive outputs or logs out of AI context. Use source declarations and schemas for dependency analysis, and neutral fixtures for reusable examples. Keep real organization names, consumer paths, and cloud identifiers out of reusable defaults and examples.
