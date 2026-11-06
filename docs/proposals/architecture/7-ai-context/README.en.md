# REP-7: Context Model for Svetoch AI Agents

[Русский](README.ru.md) · English

## Overview

Define a context model for Svetoch AI agents: where platform knowledge resides, which rules apply in a repository, and how an agent finds instructions for a specific task. **AGENTS.md defines rules and navigation; SKILL.md describes a repeatable operation**.

The first client is Codex. The agent prepares changes and runs checks; the existing CI applies infrastructure changes.

### Current State

Platform code is distributed across bazel-lib, tf-modules, and helm-charts. Documentation and REPs reside in rod-docs. Some repositories have AI instructions, but there is no shared structure or update procedure.

### Problems

- The agent has to rediscover relationships between repositories.
- Documentation may differ from the code version in use.
- There is no defined location for shared rules, module constraints, and common operations.

## Proposed Solution

Keep shared rules, the repository map, and shared skills in rod-docs. The root AGENTS.md of each working repository points to that source and requires the agent to read relevant files directly. Keep repository and local instructions alongside the code. Use skills for specific or complex tasks with a reproducible procedure. Load details as needed for the task.

### Desired State

The agent starts with the working repository's root AGENTS.md, follows its pointer to shared context in rod-docs, then locates the relevant code and reads applicable local instructions and skills.

### Benefits

A shared way of working, less repeated investigation and unnecessary context, and reuse of proven procedures.

### Trade-offs

Reading shared context requires access to rod-docs. Changes to shared rules are immediately visible to all repositories; the agent must check them against the code version in use and flag discrepancies.

## Decision log

**Date:** 25.10.2026. **Selected option:** a single shared context source in rod-docs, a pointer to it in each repository's root AGENTS.md, and instructions alongside the code.

Criteria: availability of shared context, alignment with the code version, initial context size, and maintenance cost.

| Option | Advantages | Limitations |
|---|---|---|
| All rules and procedures in each repository's root AGENTS.md | Simple setup; everything is immediately available | Unnecessary context for each task; shared rules are duplicated and diverge |
| **Pointer from the root AGENTS.md to shared context in rod-docs** | One source without copies; local instructions stay with the code; details are read as needed | Requires access to rod-docs; changes to the shared source are immediately visible to all repositories |
| Pinned local copies of shared context | Work from a single checkout with a fixed text version | Copies must be updated and checked for compatibility |

**Why the second option was selected:** shared rules and procedures change in one place. The working repository's root AGENTS.md gives the agent the exact source location and local navigation. For module behavior, the agent checks the code version in use rather than assuming that current documentation matches it.

For each task, the agent records the rod-docs commit it read in the report. This makes the context auditable without copying text or keeping a tracking file in every repository. A link to the current branch does not itself pin a version or prove compatibility with the code.

## Design Details

### 1. Shared Context Source

~~~text
rod-docs/
├── AGENTS.md                         # Rules for working in rod-docs
├── docs/ai/
│   ├── AGENTS.md                     # Shared rules for the entire platform
│   └── repositories.md               # Repository relationships and task navigation
└── .agents/skills/<operation>/
    └── SKILL.md                      # Shared procedure for an individual operation
~~~

Include the following in the shared docs/ai/AGENTS.md:

- Check versions and read the code version in use.
- Reuse existing modules and settings.
- Keep company and environment data in product configuration.
- Prohibit passing secret values, credentials, raw state, and plan into AI context.
- Run apply/deploy through CI; prohibit implicit execution of bootstrap, state operations, and cloud writes by the agent.
- Require a report of changes and checks performed.

Describe each repository's purpose, dependencies, and where to go for each type of change in repositories.md.

### 2. Working Repository Context

~~~text
<repository>/
├── AGENTS.md                         # Purpose, navigation, checks, and shared context location
└── <module>/AGENTS.md                # Additional module rules, if needed
~~~

The root AGENTS.md stores the canonical source location: `https://github.com/svetoch-dev/rod-docs/tree/master/docs/ai/`. It explicitly instructs the agent to read `docs/ai/AGENTS.md` and `docs/ai/repositories.md` from rod-docs before working, and to locate the relevant `SKILL.md` under `rod-docs/.agents/skills/` for a repeatable operation. A link provides navigation and does not load instructions into Codex context by itself. Shared files and skills are not copied into the working repository.

The root AGENTS.md defines the order: **shared rules from rod-docs → dependency versions → instructions for affected directories → applicable skill → code and checks**. It also retains the repository's purpose, local paths, and check commands. Nested instructions refine the rules but do not grant additional authority to change infrastructure. Repository-specific details live in its AGENTS.md, README, and, when needed, its own skills that do not duplicate shared procedures.

When starting from the root or a nested directory, the agent must explicitly read the shared source and applicable local files. If rod-docs is unavailable, the agent reports this and does not perform a task that requires unread shared rules or procedures.

### 3. Scope of Local Instructions

| Repository | Directories | Additional rules |
|---|---|---|
| bazel-lib | tools/, rod/, rod/scripts/init/ | Macros, generated files, Python helpers, bootstrap side effects |
| tf-modules | modules/rod/, modules/gcp/, modules/yc/ | Defaults, overrides, provider specifics, and changes to inputs/outputs |
| tf-modules | For example, modules/gcp/cloudrun_service/ | IAM, secrets, separation of configuration and image management |
| helm-charts | charts/, charts/chart_deps/app/core/, charts/chart_deps/app/common/ | Chart reuse, library/application differences, dependencies, and checks |
| helm-charts | charts/environments/, charts/environment/ | App-of-Apps, paths, revisions, CRDs, sync/pruning constraints |
| Product | Terraform, Argo CD, and local module directories | Product paths, environments, versions in use, and local specifics |

A local AGENTS.md applies to its directory and subdirectories. A separate file is needed only for additional rules; it does not override shared security constraints. Parameter descriptions remain in the module's README.

### 4. Submodules and Versions

- A Git submodule has its own root AGENTS.md, shipped with its version. Before changing a submodule, the agent explicitly reads its instructions.
- Changes to a shared library are made in its repository; the product receives the updated dependency.
- Use the gitlink commit for a submodule, and the actual refs and resolved dependency versions for Bazel and Terraform. Do not substitute the current master.
- Shared instructions and skills are read from rod-docs via the pointer in the root AGENTS.md; the rod-docs commit used is recorded in the task report. Code versions come from existing dependency files. If an instruction conflicts with the actual interface of the version in use, the agent flags it and checks the solution against the code; shared context updates are reviewed in rod-docs.

### 5. Skills: Task-Specific Procedures

A separate skill is needed for a specific or complex task that can be described as a repeatable procedure: with clear inputs, a sequence of actions, and a verifiable outcome. Shared skills live in rod-docs and are read from there when needed; the agent does not assume automatic discovery from the working repository. Skills unique to a repository may live alongside its code. A simple change already explained in README or AGENTS.md does not require a separate skill.

~~~text
.agents/skills/<name>/
├── SKILL.md          # When to apply the skill and how to perform the task
├── references/       # Details, examples, and repository specifics
├── scripts/          # Optional: helper scripts
└── assets/           # Optional: templates and starter files
~~~

YAML metadata at the beginning of SKILL.md defines **name**, the skill identifier, and **description**, the task and conditions under which the agent selects the procedure. The body contains inputs, required files and versions, steps, constraints, stop conditions, checks, and the expected outcome. SKILL.md explains which references to read and when; details are loaded as needed. Scripts and templates are added only when necessary.

Creating a skill:

1. Define the task, conditions of use, required data, and indicators of success, based on the code and existing workflow.
2. Describe actions and checks; move details to references/ and add verified scripts and templates if needed.
3. Validate the procedure against a representative task and invalid inputs: are the instructions sufficient, is the scope of changes maintained, and can the outcome be verified?

Examples illustrating skill content, rather than a required implementation list:

| Task | What to describe in the skill |
|---|---|
| Updating secrets through Terraform or Argo CD/Kubernetes | Who manages the resource, where the secret or its reference is defined, its version, permissions, and affected environments; how to prepare and check the change through the established secure process without passing the secret value to the agent |
| Upgrading Terraform/Helm dependencies | Where versions are pinned, how to check compatibility and required migrations, which related files to update, and which checks to run |
| Adding Cloud Run services | How to select an existing module, configure the service account, access, and resources, connect the service in the required environments, and check the changes |

The outcome is a diff and a list of checks. The agent prepares changes; CI applies infrastructure changes. Secret values are excluded from context. Before running a helper script, the agent checks its actions and side effects.

### 6. How Task Context Is Assembled

1. The working repository's root AGENTS.md defines its purpose, local navigation, and checks, and provides the shared source location in rod-docs.
2. The agent explicitly opens the shared rules and repository map in rod-docs, records the commit read, and identifies relationships with other platform components.
3. The agent identifies versions of affected dependencies and reads local AGENTS.md files at the location of the change.
4. For a repeatable operation, the agent opens the relevant SKILL.md in rod-docs or a local skill; its references and code are read as needed.
5. Based on this context, the agent prepares changes and checks. Inapplicable modules and procedures are not loaded in full.

Local instructions change alongside the code. Shared rules and skills change in rod-docs; working repositories contain only a pointer to the source and their own specifics. A mismatch between an instruction and the actual code interface is flagged as a discrepancy; an instruction from a newer version is not applied automatically.
