---
title: "Why I Built a Deterministic RAG System"
date: 2026-10-28
draft: false
tags: ["Architecture", "AI", "RAG"]
---

"MCP is the future, it will revolutionize how AI interacts with APIs", they say.

The hype around Agentic AI and Model Context Protocol (MCP) is deafening. Every day, we
see demos of LLMs using the MCP to browse the web, query APIs, and make autonomous
decisions. It looks like magic. But if you are building software for a critical enterprise
system or any critical industry that involves human lives or safety, magic is exactly what
you should stay away from.

## The Hype of Agentic Retrieval

Usually LLM is given set of tools (e.g., `read_file`, `search_database`). In a typical
agentic flows, user asks a question and the LLM decides which tools to call, read the
output and generates answer.

On paper, it is way more flexible, indeed. But in practice, it is much harder to build,
secure, and test. You have to handle infinite loops, latency, and the fact that LLM might
hallucinate the tool arguments are some of the things that need to be handled. But the
biggest issue is **consistency**.

With MCP, if you ask the same question twice, the LLM might decide to call different
tools, or search for different keywords. The context changes, so the answer changes.

## Predictable Retrieval

Instead of ordering the LLM to search, classic Retrieval Augmented Generation which
separates the retrieval from the generation, might works better depending on your use case
or constraint.

Let's see how classic RAG flow works. First, the user ingest data. Then the system embeds
the text into a vector database. When query comes in, the system performs a mathematical
search (cosine similarity) to fetch the top-K chunks, inject them into LLM prompt and let
the LLM generates the answer.

Retrieval is a database read. It is fast, cheap, and highly predictable. If you ask the
same question twice, the math guarantees you will retrieve the exact same chunks.

## Forcing the LLM to Behave

RAG is by no means deterministic. Because even with strict top-K filter, you still inject
the chunks into a probabilistic LLM. But we can make the system **highly reproducible** by
configuring the LLM itself:

- Set the temperature to 0 (to remove creativity)
- Fix the LLM seed (for local models)
- Freeze the vector index

When you combine deterministic retrieval with strict generation parameter, the entire
system becomes reproducible. You can audit exactly why the AI said what it said and trace
it back to the source where you or user ingests the data

_Note: Because GPUs calculate floating point math in parallel, LLMs are never 100%
deterministic across different machines, but for a single-node system, `temperature=0` and
a fixed seed are as close as we can get_

## Conclusion

AI models will never be perfectly deterministic, but it will get more accurate as
technology advances.

The rate of error might eventually drop so low people won't notice or care. But for
critical sectors involving human life that slight margin of error is intolerable.

That is exactly why predictable retrieval systems like RAG, which prioritize auditability
and exact source citations over "agentic magic", will always have a place in enterprise
software.
