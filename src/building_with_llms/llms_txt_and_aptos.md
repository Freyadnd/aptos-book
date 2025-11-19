# llms.txt and Aptos

`llms.txt` is a community convention for exposing documentation in a plain-text format that LLMs can consume efficiently.  The idea is simple: provide a `/llms.txt` file at the root of a documentation site that links to the most useful pages in a machine-readable structure.  This book is built with `mdbook-llms-txt-tools`, which generates this file automatically.

## Why It Matters

When you ask an LLM about Aptos, it draws on training data that may be months or years old.  Providing the model with a current `llms.txt` (or the full text of relevant pages) lets it give accurate, up-to-date answers without hallucinating deprecated APIs.

This is especially useful for:

- **Code generation** — the model knows the current module structure and function signatures.
- **Debugging** — the model can reference actual error codes and their meanings.
- **Documentation Q&A** — users get answers grounded in the real docs, not stale training data.

## The llms.txt Format

A minimal `llms.txt` file looks like this:

```
# Aptos Book

> The Aptos Book is a comprehensive guide to the Aptos blockchain and the Move programming language.

## Getting Started

- [Installation](https://aptos-book.com/getting_started/installation.md): How to install the Aptos CLI and set up your development environment.
- [Hello Blockchain!](https://aptos-book.com/getting_started/hello_blockchain.md): Deploy and call your first Move module.

## Move Language

- [Common Programming Concepts](https://aptos-book.com/common_programming_concepts/intro.md): Variables, functions, control flow.
- [Ownership](https://aptos-book.com/ownership/intro.md): Move's resource ownership model.
- [Structs](https://aptos-book.com/structs/intro.md): Defining and using structs and resources.
```

The format follows the [llms.txt specification](https://llmstxt.org/): a Markdown file with a title, an optional blockquote description, and sections linking to individual pages.

## How This Book Generates llms.txt

The `book.toml` for this book includes the `mdbook-llms-txt-tools` preprocessor.  When you run `mdbook build`, it produces:

- `llms.txt` — the index file with links to all chapters.
- `llms-full.txt` — the full concatenated text of every chapter, suitable for pasting directly into a model context.

Both files appear in the `book/` output directory alongside the HTML build.

## Using the Output with an LLM

**Option 1: Reference the URL**

Many LLM interfaces (Claude, ChatGPT with browsing, etc.) can fetch a URL.  Point the model at `https://aptos-book.com/llms.txt` and ask it to use the contents as context.

**Option 2: Paste the full text**

For offline or API use, paste the contents of `llms-full.txt` into the system prompt or as a user message before asking your question.

```python
import anthropic

with open("llms-full.txt") as f:
    book_text = f.read()

client = anthropic.Anthropic()

message = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=1024,
    system=f"You are an Aptos developer assistant. Use the following documentation:\n\n{book_text}",
    messages=[
        {"role": "user", "content": "How do I emit an event in Move?"}
    ],
)
print(message.content[0].text)
```

**Option 3: Tool / retrieval augmented generation**

For production agent systems, embed each page of the book and retrieve the top-k most relevant chunks at query time rather than including the entire book in every request.  This keeps token costs predictable as the documentation grows.

## Keeping Context Fresh

Documentation changes over time.  A few practices to keep the LLM's context current:

- Re-download `llms.txt` or `llms-full.txt` at the start of each session rather than caching a local copy indefinitely.
- Pin to a specific book release if you need reproducibility.
- For agents, add the book version (visible in `book.toml`) to the system prompt so the model can flag when it might be out of date.

## Further Reading

- [llmstxt.org](https://llmstxt.org) — the community specification.
- [mdbook-llms-txt-tools](https://crates.io/crates/mdbook-llms-txt-tools) — the mdBook plugin used by this book.
- [AI Agents on Aptos](ai_agents_on_aptos.md) — building agents that transact on-chain.
