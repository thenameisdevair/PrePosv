# PrePosv — Protocol Analyzer Skill
## Product Requirements Document

### What This Is
A Claude Code skill that performs pre-audit threat modeling on Solidity protocol repositories.
Invoked via `/protocol-analyzer` from inside a cloned repo. Produces `THREAT-MODEL.md` in the
repo root. That file is passed as context to pashov's `/solidity-auditor` before it scans.

### The Problem
Pashov's auditor runs cold — it applies the same generic checklist to every protocol regardless
of what the protocol actually does. A lending protocol and a bridge have completely different
threat surfaces. Pre-seeding the auditor with a protocol-specific threat model means it scans
for the right class of bugs, not all classes equally.

### Design Decisions

**Manual repo setup**
The skill always runs from inside an already-cloned repo. No git clone inside the skill.
Clean separation of concerns — the skill is a pure analysis tool.

**Dynamic question generation**
Protocol type is classified first. That classification feeds a generator that produces exactly
5 protocol-specific probing questions. Nothing is hardcoded per category — the questions are
specific to this protocol's mechanism, not just its type.

**Model allocation**
- Opus: Classification (Stage 0), Question Generation (Stage 1), Hypothesis Synthesis (Stage 3)
- Sonnet: Investigation (Stage 2), Gap Assessment (Stage 4)
High-stakes reasoning on Opus. Mechanical reading passes on Sonnet.

**Scope filtering — layered approach**
1. Read foundry.toml or hardhat.config to get declared src path
2. Path exclusions within src (test/, mocks/, script/, lib/)
3. Content detection — exclude files where all contracts are Mock/Test/Interface only
4. Import graph traversal — follow imports from confirmed core files regardless of location
5. Output file set before analysis begins for human confirmation

**Manual pashov handoff**
This skill does not trigger pashov automatically. When running pashov, the user points it
to THREAT-MODEL.md as context manually.

### Output Schema — THREAT-MODEL.md

**Section 1 — Protocol Fingerprint**
What the protocol does economically. Actors. External protocol dependencies. Value lifecycle.

**Section 2 — Asset & Trust Boundary Map**
Every asset the protocol touches. Every trust boundary. Structured list, not prose.

**Section 3 — Privileged Role Inventory**
Every role, what it controls, worst-case impact if compromised. Timelock status.

**Section 4 — Architectural Risk Flags**
Proxy patterns, delegatecall, storage layout, factory patterns. Factual, with file references.

**Section 5 — Prioritized Threat Hypotheses**
Protocol-specific falsifiable hypotheses. Each must contain: exact location, mechanism,
precondition, what breaks, exposure magnitude. Ranked Critical / High / Medium by asset exposure.
The 5 dynamically generated questions appear here as the investigation agenda.

**Section 6 — Coverage Gaps**
What could not be assessed statically. Off-chain dependencies. Missing repo components.

### Pipeline

Stage 0 — Classification (Opus)
  Read codebase → PROTOCOL_TYPE, HYBRID_COMPONENTS, CORE_MECHANISM, KEY_CONTRACTS

Stage 1 — Question Generation (Opus)
  Take Stage 0 output → generate 5 protocol-specific probing questions

Stage 2 — Investigation (Sonnet)
  Answer the 5 questions + populate Sections 1-4 via targeted reading passes

Stage 3 — Hypothesis Synthesis (Opus)
  Reason over Stage 2 findings → generate Section 5

Stage 4 — Gap Assessment (Sonnet)
  Identify static analysis limits → generate Section 6
