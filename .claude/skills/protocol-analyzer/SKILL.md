---
name: protocol-analyzer
description: Pre-audit threat modeling skill for Solidity protocol repositories.
             Automatically invoked when auditing, reviewing, or analyzing smart contracts.
             Classifies protocol type, generates protocol-specific probing questions,
             maps asset flows and trust boundaries, identifies architectural risks,
             and produces a prioritized THREAT-MODEL.md for use as pashov auditor context.
allowed-tools: Read, Write, Bash, Glob, Grep
model: claude-opus-4-6
---

## Stage 0 — Scope Filtering & Classification

Before reading any contract, establish what to read.

### Step 1 — Determine Source Path
Read `foundry.toml` if present. Extract the `src` value under `[profile.default]`.
If not present, read `hardhat.config.js` or `hardhat.config.ts` and extract the source path.
If neither exists, default to scanning: `src/`, `contracts/`, `protocol/`, `core/`.

### Step 2 — Path Exclusions
Within the identified source path, exclude:
- Any directory named: `test`, `tests`, `mocks`, `mock`, `script`, `scripts`, `lib`, `node_modules`, `out`, `artifacts`, `cache`
- Any file matching: `*.t.sol`, `*.s.sol`

### Step 3 — Content Detection
Within remaining files, exclude any file where every contract definition meets one of:
- Name is prefixed with: `Mock`, `Fake`, `Test`, `Stub`, `Helper`
- Contains `import "forge-std"` or `import "ds-test"`
- Every function body is empty or returns a hardcoded literal

### Step 4 — Import Graph Traversal
From the confirmed core files, follow every `import` statement recursively.
Add any imported file to the core set regardless of its location or name.
This catches shared libraries, abstract base contracts, and production mocks
that live outside the standard source path.

### Step 5 — Output File Set
Before proceeding, output the complete list of files that will be analyzed.
Format:
```
SCOPE CONFIRMED — FILES TO ANALYZE:
- path/to/Contract.sol
- path/to/AnotherContract.sol
...
EXCLUDED:
- path/to/MockContract.sol (reason: Mock prefix)
...
```
Pause here. If the scope looks wrong, the user can correct it before analysis begins.

### Step 6 — Classification
Read the confirmed core files. Produce the following structured output:

```
PROTOCOL_TYPE: [lending | AMM | liquid-staking | vault | bridge | perps | stablecoin | governance | hybrid]
HYBRID_COMPONENTS: [list if hybrid, otherwise none]
CORE_MECHANISM: [one sentence — what this protocol does economically]
KEY_CONTRACTS: [the 3-5 contracts containing core logic]
```

Signals to read for classification:
- Entry point functions: `deposit()`, `borrow()`, `swap()`, `mint()`, `stake()`
- Exit point functions: `withdraw()`, `repay()`, `redeem()`, `unstake()`
- Event emissions: what the protocol considers meaningful state changes
- External protocol imports: Aave, Uniswap, Chainlink integrations reveal protocol category
- README.md if present — read it but verify every claim against code

## Stage 1 — Dynamic Question Generation

Using the classification output from Stage 0, generate exactly 5 protocol-specific
probing questions. These questions drive the entire investigation in Stage 2.

### The Question Generation Prompt
Reason over the classification output using the following framework:

```
You are a senior smart contract auditor with deep expertise in DeFi protocol security.

You have classified this protocol:
PROTOCOL_TYPE: {PROTOCOL_TYPE}
HYBRID_COMPONENTS: {HYBRID_COMPONENTS}
CORE_MECHANISM: {CORE_MECHANISM}
KEY_CONTRACTS: {KEY_CONTRACTS}

Generate exactly 5 questions that probe the most critical, non-obvious failure points
specific to this protocol's design and mechanism.

Rules:
- Each question must be answerable by reading the codebase
- Each question must target a different failure domain from this list:
  [accounting | trust-boundaries | state-machine | transaction-ordering | arithmetic]
- Questions must be specific to this protocol's actual mechanism — not generic
- Frame each question around where this protocol's specific assumptions can break
- No generic questions like "is there reentrancy" — name the specific functions,
  mechanisms, and conditions relevant to this protocol
- Each question should expose a failure mode that would result in loss of funds,
  loss of protocol integrity, or violation of a core invariant

Output format:
Q1: [accounting] <question>
Q2: [trust-boundaries] <question>
Q3: [state-machine] <question>
Q4: [transaction-ordering] <question>
Q5: [arithmetic] <question>
```

### Output
Store the 5 generated questions. They will:
1. Guide the reading passes in Stage 2
2. Appear in THREAT-MODEL.md Section 5 as the investigation agenda
3. Each question must be answered explicitly before Stage 3 begins

Display the questions to the user before proceeding:
```
INVESTIGATION AGENDA — 5 PROTOCOL-SPECIFIC QUESTIONS:
Q1: [accounting] <question>
Q2: [trust-boundaries] <question>
Q3: [state-machine] <question>
Q4: [transaction-ordering] <question>
Q5: [arithmetic] <question>

Proceeding to Stage 2 — Investigation...
```
