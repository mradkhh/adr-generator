# ADR Generator

A standalone Agent Skill for recording architectural and business decisions with their rationale, consequences, and verification evidence across technology stacks, monorepos, and business-only projects. Each project keeps its own decision history. This repository is the canonical source for the skill. It is not an OpenAI-endorsed or catalog-listed product.

## Install for Codex

Clone this repository as `adr-generator`. For access across your projects, link the checkout into your user skill directory:

```bash
mkdir -p "$HOME/.agents/skills"
ln -s /absolute/path/to/adr-generator "$HOME/.agents/skills/adr-generator"
```

For a project-scoped installation instead, link it into that project's skill directory. Substitute absolute paths for your checkout and project:

```bash
mkdir -p /absolute/path/to/project/.agents/skills
ln -s /absolute/path/to/adr-generator /absolute/path/to/project/.agents/skills/adr-generator
```

Use the installation scope you need; avoid installing the same skill in both locations for the same project. The destination must not already exist. Inspect an existing installation before replacing it. A symlink uses your local checkout directly; its absolute target is machine-specific. For a portable checked-in installation, copy `SKILL.md`, `agents/`, and `assets/` together into `.agents/skills/adr-generator/` and record the source commit when updating the copy.

The folder name must remain `adr-generator`. The skill at this repository's root is its distributable source; cloning it alone does not install it into another project's discovery path. If discovery does not refresh, restart Codex. See [Codex skill discovery](https://learn.chatgpt.com/docs/build-skills).

Other clients that support the [Agent Skills format](https://agentskills.io/specification) can use the same folder through their own installation mechanism. `agents/openai.yaml` is Codex-specific metadata; compatibility with every client is not claimed. The skill needs file access and uses Git for Git-based requests. It requires no custom runtime, MCP server, or network access for ordinary local documentation.

## Use

From the target project, ask for a bounded decision record:

```text
$adr-generator Document the architectural and business decisions in the
uncommitted and unpushed changes. Use docs/implementation/contracts.md
as intent, compare it with the code, and record verification gaps.
```

```text
$adr-generator Record the agreed business rule that an order's price is
fixed at checkout. The product owner accepted this rule today; no code
implementation is claimed by this request.
```

```text
$adr-generator Record the accepted replacement for ADR 003 using the
decision and verification evidence in this conversation. Preserve ADR 003.
```

Relevant natural-language ADR requests can also activate the skill. Ordinary implementation and changelog requests alone should not.

You can name the project and destination explicitly in natural language:

```text
$adr-generator Document the accepted offline-storage decision for
/projects/mobile. Write the ADR into documentation/decisions.
Use the implementation and verification evidence in this conversation.
```

This selects `/projects/mobile/documentation/decisions/`, even when invoked from another directory. These are prompt instructions, not CLI flags.

## Where documents are collected

Installing the skill globally makes the workflow available across projects; it does not collect their documents in a global folder. The project root comes from an explicit user-selected project, otherwise the Git root containing the working directory, otherwise the working directory for a project without Git. An ambiguous parent folder containing several projects requires selecting the target project first.

The output directory follows this precedence:

1. The user's explicit destination for the task.
2. Applicable project documentation rules, such as `AGENTS.md`.
3. An existing ADR catalog appropriate to the decision scope.
4. `<project-root>/docs/adr/`.

Relative output paths are based on the selected project root; absolute paths are honored as supplied. An explicitly documented alternative path base in project instructions is respected. Multiple plausible catalogs are clarified before writing. Existing records are not moved when a different destination is selected. A missing catalog is created when an ADR is ready; write failures are reported rather than redirected to another folder.

```text
~/.agents/skills/adr-generator/       Shared skill installation

/projects/backend/docs/adr/          Backend decision history
    001-authentication.md
    002-order-pricing.md

/projects/mobile/docs/adr/           Mobile decision history
    001-offline-storage.md
```

A monorepo defaults to one root catalog. A service-specific catalog is used only when project conventions or the request establish it. Numbering and duplicate-ID checks are local to the selected catalog; separate projects can both start at `001`. Cross-catalog references identify the actual file through a relative path, not just its number.

Before writing, the agent reports the selected project, absolute ADR directory, and selection basis. This does not require routine confirmation. The final response links to the actual output file. Documents go into the skill repository only when the skill repository itself is the project being documented.

## Output and history

Records in the selected catalog capture context, decision, consequences, verification, and references. A new catalog starts at `001`; later records use the largest existing numeric prefix in that catalog plus one.

Drafts can be refined. Finalized decisions retain their historical content; a later record links back when a decision is replaced or retired. Under the default append-only convention the earlier status remains historical, so read related later records to determine the current decision. Existing explicit project lifecycle conventions take precedence.

When an existing rule changes, a `Change` subsection records `Before`, `After`, and `Reason`. A replacement uses a `Supersedes` metadata field linking to the actual prior ADR, regardless of its number. Partial replacements name the affected rules and preserve the remainder. A first decision needs neither a replacement field nor a comparison with a fictional earlier rule.

For example, a replacement record might contain:

```markdown
- **Supersedes:** [ADR 003](003-order-pricing.md)

### Change

- **Before:** ADR 003 recalculates prices from the catalog until dispatch.
- **After:** Freeze order prices at checkout.
- **Reason:** Preserve the price agreed by the customer at checkout.
```

The example assumes the rationale is supported by the supplied decision. In a Proposed record the replacement is only intended; the earlier Accepted rule remains effective. A larger ADR number does not establish precedence.

Before writing, the agent checks duplicate IDs, missing replacement targets, self-references, replacement cycles, and conflicting Accepted rules in the relevant scope. Ordinary references and valid numbering gaps are allowed. Ambiguous history is reported with file references rather than silently repaired. These are agent instructions, not an automated runtime validator.

`Accepted` requires an acceptance basis, and claims of completed implementation require relevant verification evidence. Business decisions do not need fictional code tests. Acceptance does not prove deployment. Missing evidence is stated explicitly. The skill documents consequential decisions rather than producing one ADR for every commit.

## Maintain and validate

Keep the workflow in `SKILL.md`, the output shape in `assets/adr-template.md`, and Codex presentation metadata in `agents/openai.yaml`. No generator script or runtime dependencies are shipped.

When the Codex `skill-creator` tooling is available, run its validator against this folder:

```bash
python /path/to/skill-creator/scripts/quick_validate.py /absolute/path/to/adr-generator
```

The Agent Skills reference validator is another format check when installed:

```bash
skills-ref validate /absolute/path/to/adr-generator
```

Format validation does not establish correct decisions. Exercise these requests in a temporary project and inspect the generated records:

| Scenario | Expected result |
| --- | --- |
| Empty catalog and a supported decision | Complete `001-...md` record |
| Existing `001` and `003` | Next record is `004`, with no historical edits |
| Required tests skipped or unavailable | Proposed; no fabricated successful verification |
| Plan contradicts current source | Discrepancy is explicit; status reflects missing required evidence |
| Accepted decision replaced | New record has Supersedes and Before/After/Reason; previous record stays unchanged |
| First decision with no previous rule | No Supersedes field or invented Before state |
| Only wording or implementation detail changes | Existing decision returned; no filler ADR |
| Partial replacement | Changed rules are explicit; unaffected rules remain in force |
| Proposed replacement of an Accepted rule | Original rule remains effective |
| Duplicate IDs or missing replacement target | Defect reported; ambiguous affected record is not written |
| Self-reference or circular replacement chain | Invalid relationship reported, including cycles introduced by a new record |
| Conflicting Accepted rules without effective replacement | Conflict reported; larger number does not win |
| Ordinary cross-references or distinct decision scopes | No false replacement-cycle or semantic-conflict finding |
| Accepted business-only decision | Acceptance basis stated; no invented code or test results |
| Same decision already recorded | Existing record returned without a duplicate |
| Non-CRM project or no Git history | Project-local output; no CRM assumptions or invented Git range |
| Invocation from a repository's src/ subdirectory | Repository-root catalog is selected unless explicitly overridden |
| Monorepo with no service-specific convention | One root catalog; no automatic package catalogs |
| Explicit subproject or service-specific convention | Selected scope is respected |
| Relative custom output with an explicit project | Destination is resolved from that project root |
| Existing nonstandard ADR directory | Existing catalog is reused without a parallel docs/adr/ |
| Ambiguous parent folder or multiple plausible catalogs | Target is clarified before writing |
| Two projects both containing ADR 001 | Numbering is independent; no false duplicate finding |
| Global skill installation used from another project | Output belongs to the target project, not the skill checkout |
| Unwritable selected directory | Destination and failure reported; no silent fallback |

Check description matching with both ADR requests and unrelated editing/changelog requests. Check local links, unresolved placeholders, number collisions, and the final file changes. A clean validator result is only a format result.

## Origin

Adapted from the CRM `generate-adr` skill's Architecture and Business Decision Documentation System. It preserves durable decision history and makes evidence, status, and replacement rules explicit. The old CRM installation is not automatically modified or synchronized; migrate it separately to avoid two overlapping skill descriptions.

The workflow also takes inspiration from migration history: compare a relevant prior state, record the change, and check historical relationships. Each ADR carries its own evidence and replacement links; no separate journal or full architecture snapshots are maintained.

The package follows the [Agent Skills specification](https://agentskills.io/specification) and uses the optional metadata documented in [OpenAI's skill guide](https://learn.chatgpt.com/docs/build-skills). No license grant is declared by this initial package.
