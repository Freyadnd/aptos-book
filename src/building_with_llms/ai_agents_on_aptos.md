# AI Agents on Aptos

An AI agent is a program that perceives its environment and takes actions autonomously.  On a blockchain, that means reading chain state and submitting transactions without a human approving each one.  Aptos is well-suited for agent workloads because of its high throughput, low finality time, and expressive TypeScript and Python SDKs.

## What Makes Aptos a Good Agent Platform

- **Sub-second finality** — agents get confirmation quickly and can chain actions without long waits.
- **Parallel execution** — multiple agents can transact simultaneously without contending on global state (see [Parallelization](../parallelization/intro.md)).
- **View functions** — `#[view]` entry points let agents read contract state without paying gas.
- **Typed events** — structured `#[event]` emissions give agents a clean feed of what happened on-chain.

## Agent Architecture

A minimal Aptos agent has three parts:

```
┌─────────────────────────────────────────────┐
│  Perception layer                           │
│  (read chain state, watch events, get data) │
└────────────────────┬────────────────────────┘
                     │
┌────────────────────▼────────────────────────┐
│  Decision layer                             │
│  (LLM or rule engine — what to do next)     │
└────────────────────┬────────────────────────┘
                     │
┌────────────────────▼────────────────────────┐
│  Action layer                               │
│  (build and submit Aptos transactions)      │
└─────────────────────────────────────────────┘
```

## Reading Chain State

Use the Aptos TypeScript SDK to query view functions and account resources:

```typescript
import { Aptos, AptosConfig, Network } from "@aptos-labs/ts-sdk";

const config = new AptosConfig({ network: Network.MAINNET });
const aptos = new Aptos(config);

// Read a resource from an account
const resource = await aptos.getAccountResource({
  accountAddress: "0xabc123...",
  resourceType: "0x1::coin::CoinStore<0x1::aptos_coin::AptosCoin>",
});

// Call a view function
const [result] = await aptos.view({
  payload: {
    function: "my_addr::counter::get",
    functionArguments: ["0xabc123..."],
  },
});
```

## Submitting Transactions

Agents need a funded account and a private key.  Keep the private key in an environment variable, not in source code.

```typescript
import { Account, Ed25519PrivateKey } from "@aptos-labs/ts-sdk";

const privateKey = new Ed25519PrivateKey(process.env.AGENT_PRIVATE_KEY!);
const agent = Account.fromPrivateKey({ privateKey });

const transaction = await aptos.transaction.build.simple({
  sender: agent.accountAddress,
  data: {
    function: "my_addr::counter::increment",
    functionArguments: [],
  },
});

const committed = await aptos.signAndSubmitTransaction({
  signer: agent,
  transaction,
});

await aptos.waitForTransaction({ transactionHash: committed.hash });
```

## Watching Events

Agents often need to react to on-chain events.  Poll the indexer or use a GraphQL subscription:

```typescript
// Fetch recent events for a module
const events = await aptos.getModuleEventsByEventType({
  eventType: "my_addr::my_module::TransferEvent",
  minimumLedgerVersion: BigInt(lastProcessedVersion),
});

for (const event of events) {
  await handleEvent(event);
}
```

> Store `lastProcessedVersion` durably so the agent can resume without replaying old events after a restart.

## Connecting an LLM Decision Layer

The decision layer receives a description of the current state and outputs an action.  A minimal loop using the Claude API:

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();

async function decideAction(state: string): Promise<string> {
  const message = await client.messages.create({
    model: "claude-opus-4-6",
    max_tokens: 256,
    messages: [
      {
        role: "user",
        content: `You are an Aptos DeFi agent. Current state:\n${state}\n\nWhat should I do next? Reply with one of: HOLD, BUY, SELL, or WAIT.`,
      },
    ],
  });

  const text = message.content[0];
  if (text.type === "text") return text.text.trim();
  return "WAIT";
}
```

Keep the decision prompt narrow and the output format constrained.  Structured outputs and tool use in the Claude API make it easier to parse the model's response programmatically.

## Safety Considerations

Autonomous agents submitting transactions carry real risk.  A few principles:

- **Set a spending limit** — track cumulative spend in a session and halt if it exceeds a threshold.
- **Dry-run first** — use `aptos.transaction.simulate` to estimate gas and check for errors before submitting.
- **Rate limit actions** — add a minimum delay between transactions to prevent runaway loops.
- **Log everything** — record every decision and transaction hash so you can audit what happened.
- **Test on devnet first** — Aptos devnet is free and has the same API as mainnet.

## Further Reading

- [Aptos TypeScript SDK](../tools/ts-sdk/intro.md)
- [Events](../events/intro.md)
- [Parallelization](../parallelization/intro.md)
