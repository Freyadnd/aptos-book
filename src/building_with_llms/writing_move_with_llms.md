# Writing Move with LLMs

LLMs can generate Move code, explain unfamiliar patterns, and suggest idiomatic rewrites.  Getting the most out of them requires giving the model enough context about the Aptos platform and Move's ownership model.

## Providing Context

Move differs from Solidity and other smart contract languages in ways that trip up models trained mostly on EVM code.  A few things worth stating explicitly in your prompt:

- Aptos uses an **account-centric resource model** — resources live under an account's address, not inside a contract.
- Move enforces **linear types** — a value with the `key` or `store` ability cannot be silently duplicated or dropped.
- Entry functions receive a `&signer` argument that represents the transaction sender; there is no `msg.sender` global.
- Gas is charged for execution, I/O, and storage separately.  See [Gas](../gas/intro.md) for details.

Example system prompt addition:

```
You are helping me write Move smart contracts for the Aptos blockchain.
Aptos uses a resource-oriented model where structs with the `key` ability
are stored under an address via `move_to` and retrieved with `borrow_global`
or `move_from`.  Do not generate Solidity or EVM patterns.
```

## A Practical Workflow

1. **Describe the invariant, not just the feature.**  Tell the model what must always be true (e.g., "a user can only have one `Vault` resource at a time") so it picks the right storage pattern.
2. **Ask for the struct definitions first.**  Once the data model looks right, ask for the function bodies.
3. **Request inline comments** that explain why each line is written the way it is.  This makes review faster.
4. **Iterate with the compiler.**  Paste `aptos move compile` errors back into the conversation — the model can usually fix them in one round.

## Example: Generating a Simple Counter

Here is a prompt and the expected shape of output:

**Prompt:**
```
Write an Aptos Move module called `counter` under the address `my_addr`.
It should let any account create their own counter, increment it, and read it.
Use the resource model so each account stores their own Counter struct.
```

**Expected output:**
```move
module my_addr::counter {
    use std::signer;

    struct Counter has key {
        value: u64,
    }

    public entry fun create(account: &signer) {
        let addr = signer::address_of(account);
        assert!(!exists<Counter>(addr), 1);
        move_to(account, Counter { value: 0 });
    }

    public entry fun increment(account: &signer) acquires Counter {
        let addr = signer::address_of(account);
        let counter = borrow_global_mut<Counter>(addr);
        counter.value = counter.value + 1;
    }

    #[view]
    public fun get(addr: address): u64 acquires Counter {
        borrow_global<Counter>(addr).value
    }
}
```

> Always verify generated code by running `aptos move test` before deploying.  LLMs can produce plausible-looking code that compiles but violates invariants.

## Common Mistakes to Watch For

| Mistake | Why it matters |
|---|---|
| Missing `acquires` annotation | The Move VM requires explicit declaration of global resources accessed in a function. |
| Dropping a resource silently | Resources with `key` must be explicitly destroyed with `move_from` and destructured. |
| Using `copy` on a resource | Resources cannot be copied; only structs without `key` or `store` that have `copy` can be. |
| EVM-style event emission | Aptos uses `#[event]` structs and `aptos_framework::event::emit`, not Solidity events. |

## Asking for Tests

Ask the model to generate unit tests at the same time as the module:

```
Also generate Move unit tests for this module using #[test] and #[test_only].
Cover the happy path and the case where a user tries to create a counter twice.
```

See [How to Write Tests](../testing/intro.md) for the testing framework details.
