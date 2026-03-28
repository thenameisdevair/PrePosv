# PrePosv — Protocol Analyzer

A Claude Code skill that performs pre-audit threat modeling on Solidity protocol repositories.

## What It Does

Runs before pashov's `/solidity-auditor`. Produces a `THREAT-MODEL.md` that primes the
auditor to scan for protocol-specific threats rather than running a generic checklist.

A lending protocol and a bridge have completely different threat surfaces. Running pashov
cold means it applies the same questions to both. PrePosv classifies the protocol first,
generates five targeted probing questions specific to that protocol's mechanism, maps every
asset flow and trust boundary, identifies architectural risks, and synthesizes prioritized
threat hypotheses — all before pashov scans a single line.

## How To Use

**1. Clone the target protocol repo and navigate into it**
```bash
git clone https://github.com/<target>/<protocol>
cd <protocol>
```

**2. Invoke the skill**
```bash
/protocol-analyzer
```

**3. Review the scope**
The skill outputs the files it will analyze before proceeding. Verify the scope is correct.

**4. Wait for THREAT-MODEL.md**
The skill runs five stages and writes THREAT-MODEL.md to the repo root.

**5. Pass THREAT-MODEL.md to pashov**
When invoking pashov's solidity-auditor, reference the threat model as context so it
scans for the threats specific to this protocol.

## The Pipeline

| Stage | Model | What Happens |
|-------|-------|--------------|
| Stage 0 | Opus | Scope filtering + protocol classification |
| Stage 1 | Opus | Generates 5 protocol-specific probing questions |
| Stage 2 | Sonnet | 5 reading passes — maps asset flows, roles, architecture, answers questions |
| Stage 3 | Opus | Synthesizes prioritized threat hypotheses |
| Stage 4 | Sonnet | Gap assessment + writes THREAT-MODEL.md |

## Output Schema

```
THREAT-MODEL.md
├── Section 1 — Protocol Fingerprint
├── Section 2 — Asset & Trust Boundary Map
├── Section 3 — Privileged Role Inventory
├── Section 4 — Architectural Risk Flags
├── Section 5 — Prioritized Threat Hypotheses
└── Section 6 — Coverage Gaps
```

## Installation

This skill is designed for global scope — available across every project.

```bash
mkdir -p ~/.claude/skills/protocol-analyzer
cp SKILL.md ~/.claude/skills/protocol-analyzer/SKILL.md
```

## Related Tools

- [pashov/solidity-auditor](https://github.com/pashov/audits) — downstream consumer of THREAT-MODEL.md
- [DeTest](https://github.com/thenameisdevair/DeTest) — proves or disproves audit findings via Foundry tests
- [SPECTRA](https://github.com/thenameisdevair/SPECTRA) — autonomous auditing agent

## Design Decisions

Full reasoning behind every architectural decision is documented in [PRD.md](./PRD.md).
