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

## Stage 2 — Investigation

Model: Sonnet (mechanical reading passes)

Using the 5 questions from Stage 1 as the investigation agenda, perform targeted reading
passes across the core file set. Simultaneously populate Sections 1-4 of THREAT-MODEL.md.
Answer every question explicitly before Stage 3 begins.

### Reading Pass 1 — Protocol Fingerprint (Section 1)
Target: README.md, entry/exit functions, event definitions

- Read README.md. Extract the protocol's own description. Flag any claim that
  contradicts what the code actually does.
- Grep for: `payable`, `deposit`, `borrow`, `mint`, `stake`, `swap`
  → these are value entry points. For each: who calls it, what asset enters, what is returned
- Grep for: `withdraw`, `redeem`, `claim`, `exit`, `repay`, `unstake`
  → these are value exit points. For each: who calls it, what asset exits, under what condition
- Grep for: `event ` declarations
  → list every event. The event set reveals what the protocol considers meaningful state changes
- From the above: identify all actors (depositor, liquidator, keeper, LP, guardian, etc.)
- From the above: identify all external protocol dependencies (Aave, Uniswap, Chainlink, etc.)

Output: Section 1 — Protocol Fingerprint

### Reading Pass 2 — Asset & Trust Boundary Map (Section 2)
Target: token interactions, external calls, constructor arguments

- Grep for: `IERC20`, `ERC20`, `SafeERC20`, `ERC721`, `ERC1155`
  → every asset type the protocol touches
- Grep for: `.transfer(`, `.transferFrom(`, `.safeTransfer(`, `.safeTransferFrom(`, `.call{value:`
  → every asset movement. For each: caller, condition, recipient, amount source
- Grep for: `interface I` declarations and every external call made to them
  → every trust boundary. For each: what function is called, what is done with the return value
- Grep for: constructor arguments and immutable/constant addresses
  → these are hardcoded trust assumptions
- Grep for: `setOracle`, `setPool`, `setRouter`, or any setter that changes an external address
  → these are runtime trust assumptions that can be changed

For each trust boundary output:
```
[ASSET FLOW] source → function → destination | condition
[EXTERNAL TRUST] contract → function called → how return value is used
```

Output: Section 2 — Asset & Trust Boundary Map

### Reading Pass 3 — Privileged Role Inventory (Section 3)
Target: access control declarations, modifiers, role-gated functions

- Grep for: `Ownable`, `AccessControl`, `onlyOwner`, `onlyRole`, `onlyAdmin`
  → identifies the permission system in use
- Grep for: `bytes32.*ROLE`, `keccak256.*ROLE`
  → enumerates all named roles
- For each role found: grep the modifier or role name across all files
  → list every function gated by this role
- For each gated function: read what it does
  → not just the name — the actual state changes or asset movements it enables
- Grep for: `TimelockController`, `timelock`, `delay`, `eta`
  → determines if privileged actions are delayed or immediate
- Grep for: `transferOwnership`, `grantRole`, `revokeRole`, `renounceRole`
  → who controls role management and under what conditions

For each role reason through: if this role calls every function it controls in the worst
possible sequence — what is the maximum damage? That reasoning becomes the threat statement.

Output: Section 3 — Privileged Role Inventory

### Reading Pass 4 — Architectural Risk Flags (Section 4)
Target: proxy patterns, delegatecall, storage layout, factory patterns

- Grep for: `delegatecall`
  → if present: where, with what target, is the target controllable
- Grep for: `Proxy`, `Upgradeable`, `initialize`, `_implementation`, `UUPS`, `Transparent`, `Beacon`
  → identify proxy pattern. Check if initialize() has initializer modifier guard
- Grep for: `assembly`
  → read every assembly block. Flag any direct storage slot manipulation
- Grep for: `uint256[` in storage declarations
  → gap variables signal upgrade-awareness. Their absence in upgradeable contracts is a flag
- Grep for: `CREATE2`, `clone`, `cloneDeterministic`, `Clones`
  → factory patterns. Who calls the factory, what bytecode is deployed
- Check inheritance chain of every KEY_CONTRACT
  → does any contract inherit from both upgradeable and non-upgradeable versions of the same base
- Grep for: `block.chainid`, `LayerZero`, `Wormhole`, `Axelar`, `bridge`
  → cross-chain components

Output: Section 4 — Architectural Risk Flags (short bullet list, factual, file + line references)

### Question Answering
After all four reading passes, explicitly answer each of the 5 questions from Stage 1.
Format:
```
Q1: [accounting] <question>
A1: <explicit answer with contract name, function name, and relevant code reference>

Q2: [trust-boundaries] <question>
A2: <explicit answer>
...
```

All findings from the reading passes and question answers feed directly into Stage 3.
Do not generate hypotheses yet — only report what the code shows.
