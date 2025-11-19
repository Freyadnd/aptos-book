# Auditing with LLMs

LLMs can serve as a fast first-pass security review before you bring in a human auditor.  They are good at spotting common vulnerability classes, explaining what a piece of code actually does, and suggesting defensive rewrites.  They are not a substitute for a formal audit on high-value contracts.

## What LLMs Are Good At

- **Spotting known vulnerability patterns** — reentrancy (less relevant in Move, but still possible via callbacks), integer overflow/underflow, unchecked arithmetic, missing access control.
- **Explaining unfamiliar code** — "What does this function do and what are its preconditions?"
- **Suggesting defensive additions** — missing `assert!` guards, events that should be emitted for observability, error codes that are too generic.
- **Checking invariant preservation** — "Does this function maintain the invariant that total_supply equals the sum of all balances?"

## What LLMs Are Not Good At

- Verifying complex cryptographic logic.
- Catching subtle race conditions in parallel execution paths (see [Parallelization](../parallelization/intro.md)).
- Guaranteeing absence of vulnerabilities — they can miss things, especially in novel codebases.

## A Review Workflow

### 1. Start with a Summary Request

Paste the module and ask:

```
Summarize what this Move module does, who can call each public function,
and what resources it creates or modifies.
```

This forces the model to build a mental model of the code before you ask security questions.

### 2. Ask Targeted Security Questions

```
Does this module have any missing access control checks?
Can a caller drain funds they do not own?
Are there any integer overflow risks given Move's u64 arithmetic?
```

### 3. Request a Structured Finding Report

```
Review the following Move module for security issues.
For each issue found, provide:
- Severity (Critical / High / Medium / Low / Informational)
- Location (function name and line if possible)
- Description of the issue
- Recommended fix
```

## Move-Specific Vulnerability Classes to Ask About

**Unauthorized resource access**

```move
// Bad: anyone can borrow another account's resource if the function is public
public fun withdraw_all(victim: address): Coin acquires Vault {
    let vault = move_from<Vault>(victim);
    vault.coin
}
```

Ask the model: *"Does this function verify that the caller owns the resource it is modifying?"*

**Missing error codes**

Overly generic abort codes make debugging and monitoring harder.  Ask:

```
Are all abort codes well-named constants rather than raw integers?
```

**Reentrancy via generics or callbacks**

Move does not have the same reentrancy model as Solidity, but hot-potato patterns and generic function parameters can create unexpected re-entry.  Ask:

```
Does this module accept function parameters or generic type arguments
that could call back into this module in an unexpected order?
```

## Combining LLM Review with Formal Verification

For critical modules, use the [Move Prover](../testing/formal_verification.md) to formally verify key invariants after the LLM review surfaces candidates.  The LLM can help you write the `spec` blocks:

```
Write Move Prover specification blocks for this module that verify:
1. The total supply never decreases unexpectedly.
2. Only the resource owner can call withdraw.
```

> Treat LLM audit findings as a checklist, not a verdict.  Confirm each finding manually and add a regression test before marking it resolved.
