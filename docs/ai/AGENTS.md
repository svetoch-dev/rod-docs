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
