# REP 0: Rod Enhancement Proposals (REP) Process

## Overview

This document establishes the Rod Enhancement Proposal (REP) process — a lightweight, structured workflow for proposing, discussing, and tracking significant changes to the rod project.

### Current State

Changes and enhancements to rod are currently discussed in an ad-hoc manner through issues, pull requests, and informal conversations. While this works for minor fixes, larger architectural decisions, feature additions, and process changes lack a standardized way to be proposed, reviewed, and recorded.

### Problems

- **No formal tracking**: Important design decisions are scattered across issues, PRs, and chat threads, making them hard to find later.
- **Inconsistent proposals**: Different contributors use different formats when suggesting enhancements, leading to missing context and incomplete evaluations.
- **Unclear ownership**: It is often unclear who is responsible for driving a proposal forward or who has the authority to approve it.
- **No decision history**: Once a decision is made, there is no central record of why it was made, making future reversals or re-evaluations difficult.

## Proposed Solution

Introduce a Rod Enhancement Proposal (REP) process modeled after established practices (e.g., Python PEPs, Kubernetes KEPs). Every significant change must be documented as a REP before implementation begins.

### Desired State

- A single, predictable location (`docs/proposals/`) contains all active and historical proposals.
- Every proposal follows a uniform template and metadata schema.
- Each proposal has clear ownership (authors), review (reviewers), and approval (approvers) assignments.
- Proposals move through well-defined states, visible at a glance.

### Benefits

- **Transparency**: All stakeholders can see what is being proposed, why, and by whom.
- **Consistency**: A shared template ensures every proposal includes necessary context.
- **Accountability**: Authors, reviewers, and approvers are explicitly named.
- **Traceability**: Future contributors can understand past decisions by reading archived REPs.

### Trade-offs

- **Overhead**: Small changes may feel bureaucratic. For this reason, trivial bug fixes and documentation typos do not require a REP.
- **Maintenance**: Someone must periodically archive or update stale proposals.

## Design Details

### When to Create a REP

A REP is required for:
- New major features or subsystems
- Breaking changes to public APIs or behavior
- Changes to build, release, or contribution processes
- Architectural refactoring that affects multiple components

A REP is **not** required for:
- minor features
- Bug fixes
- Documentation corrections
- dependency updates
- Config-only changes

### Directory Layout

Each REP lives in its own directory under `docs/proposals/<component>`:

```
docs/proposals/
├── 0-rep/                 # This meta-proposal
│   ├── README.md
│   └── rep.yaml
├── NNNN-rep-template/     # The official template
│   ├── README.md
│   └── rep.yaml
└──<component>/<number>-<short-name>/ # Individual proposals
    ├── README.md
    └── rep.yaml
```

`<component>` - is a rod component like 
* argocd
* tf
* init
* configuration
* architecture
* monitoring
* logging
* etc

### Metadata (`rep.yaml`)

Every REP includes a `rep.yaml` file with the following fields:

| Field | Description |
|-------|-------------|
| `title` | Short, descriptive title |
| `rep-number` | Unique integer (monotonically increasing) |
| `authors` | GitHub handles of the proposal authors |
| `component` | Affected subsystem or area |
| `status.name` | One of: `implementable`, `implemented`, `rejected`, `withdrawn`, `replaced` |
| `status.emoji` | Visual indicator: 🟢, ✅, ❌, ↩️, 🔁 |
| `creation-date` | Date the REP was created (`dd.mm.yyyy`) |
| `reviewers` | GitHub handles of assigned reviewers |
| `approvers` | GitHub handles of people with approval authority |

### Proposal Lifecycle

1. **Draft** — Author creates a new directory from the template and opens a pull request.
2. **Review** — Reviewers provide feedback. The author iterates.
3. **Approval** — Approvers merge the PR, moving the REP to `implementable`.
4. **Implementation** — Work proceeds. Once complete, status moves to `implemented`.
5. **Closure** — If rejected, withdrawn, or replaced, the corresponding status is set and the REP is archived.

### Template (`NNNN-rep-template`)

The official template contains:

- `README.md` with sections:
  - `## Overview` — Summary, current state, and problems
  - `## Proposed Solution` — Desired state, benefits, and trade-offs
  - `## Decision log` — Optional tool or library comparisons
  - `## Design Details` — Technical specifics

- `rep.yaml` — Metadata template with placeholders

Proposals **must** retain this structure so that readers always know where to find specific information.

### Numbering

- REP numbers are assigned sequentially starting from `0`.
- `0` is reserved for this process document.
- `NNNN` is the placeholder used in the template.
- Once assigned, a number never changes, even if the title does.
