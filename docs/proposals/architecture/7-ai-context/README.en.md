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

Separate instructions into shared, repository, and local levels. Use skills for specific or complex tasks with a reproducible procedure. Load details as needed for the task.

### Desired State

The agent starts with the root AGENTS.md, locates the relevant code, and reads the instructions and skill applicable to the task.

### Benefits

A shared way of working, less repeated investigation and unnecessary context, and reuse of proven procedures.

### Trade-offs

Local copies increase maintenance effort. Their provenance is recorded, and compatibility with the code version in use is checked when they are updated.

## Decision log

**Date:** 25.10.2026. **Selected option:** shared sources in rod-docs, pinned local copies, and instructions alongside the code.

Criteria: availability from the working repository, alignment with the code version, initial context size, and maintenance cost.

| Option | Advantages | Limitations |
|---|---|---|
| All rules and procedures in a single root AGENTS.md | Simple setup; everything is immediately available | Unnecessary context for each task; shared rules are duplicated and diverge between repositories |
| Read all shared context directly from rod-docs | One source; no local copies | Requires access to a separate source; the correct documentation version must be ensured for every read |
| **Shared sources in rod-docs → selected local copies → local AGENTS.md files and skills** | Work from a single checkout; pinned source; details are read as needed for the task | Copies must be updated and checked for compatibility |

**Why the third option was selected:** platform components and products are updated independently. The agent needs instructions for the code and local settings in use, available from the working repository. The shared source maintains consistent rules, while separate files allow the agent to read only the details it needs.

The cost is explicit updates to the copies. The SHA records their provenance and enables reproducibility, but does not by itself prove compatibility with the code.

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
├── AGENTS.md                         # Purpose, navigation, and check commands
├── docs/ai/
│   ├── AGENTS.md                     # Copy of shared rules from rod-docs
│   ├── repositories.md               # Copy of the shared map
│   └── sources.yaml                  # Repositories and SHAs of the copies' sources
├── .agents/skills/<operation>/
│   ├── SKILL.md                      # Selected shared skill
│   └── references/repository.md      # Paths and details specific to this repository
└── <module>/AGENTS.md                # Additional module rules
~~~

Shared copies come from rod-docs. The root AGENTS.md and references/repository.md belong to the working repository and are preserved when shared files are updated. sources.yaml stores the mapping “copy path → source repository, path, and SHA”. It is a tracking file for the model, not a built-in Codex setting.

The root AGENTS.md defines the order: **shared rules → dependency versions → instructions for affected directories → skill → code and checks**. Nested instructions refine the rules but do not grant additional authority to change infrastructure.

The shared docs/ai/AGENTS.md does not become global because of its name: the root file explicitly includes it. Links do not load instructions by themselves. When starting from the root or a nested directory, the agent must read the shared rules and applicable local files.

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
- sources.yaml records the source SHA of shared instructions and skills. Code versions come from existing dependency files. Updates to copies undergo review and compatibility checks.

### 5. Skills: Task-Specific Procedures

A separate skill is needed for a specific or complex task that can be described as a repeatable procedure: with clear inputs, a sequence of actions, and a verifiable outcome. A simple change already explained in README or AGENTS.md does not require a separate skill.

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

1. The root AGENTS.md defines the repository's purpose, navigation, and checks.
2. Shared rules and the map establish relationships with other platform components.
3. The agent identifies the versions of affected dependencies and reads local AGENTS.md files at the location of the change.
4. A skill is selected for a repeatable operation; its references and code are read as needed.
5. Based on this context, the agent prepares changes and checks. Inapplicable modules and procedures are not loaded in full.

Local instructions change alongside the code. Shared files and skills have a single source in rod-docs and are delivered through explicit updates with a recorded SHA. A mismatch between an instruction and the actual code interface is flagged as a discrepancy; an instruction from a newer version is not adopted automatically.
