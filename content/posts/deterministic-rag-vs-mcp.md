---
title: "Why I Built My Own Local RAG Pipeline Instead of Using an AI Agent"
date: 2026-10-28
draft: false
description:
  "Why I built a local RAG pipeline instead of using AI agents, and why predictable
  retrieval is critical for enterprise systems."
ShowDescription: false
tags: ["Architecture", "AI", "RAG", "MCP"]
---

<p align="center">
  <img src="/ThisIsFineGIF.gif" alt="This is fine" width="100%" />
  <br>
</p>

"MCP is the future, it will revolutionize how AI interacts with APIs", they say.

The hype around Agentic AI and Model Context Protocol (MCP) is deafening. Every day, we
see demos of LLMs using MCP to browse the web, query APIs, and make autonomous decisions.
It looks like magic. But if you are building software for a critical enterprise system or
any critical industry involving human safety, magic is exactly what you should stay away
from.

## The Agentic Search Trade-Off

In a typical agentic flow, an LLM is given tools (e.g., standard SQL queries, or keyword
search tools like `grep`). If it doesn't find the answer on the first try, the agentic
loop allows it to refine its keywords and search again. Agentic workflows actually perform
best when leveraging standard keyword search, or when skipping retrieval entirely by
dumping data into massive context windows.

On paper, this iterative loop is highly flexible, indeed. But in practice, it is much
harder to secure, test, and audit. You have to handle infinite loops, latency spikes, and
the fact that the LLM might hallucinate tool arguments [^1]. Most importantly, it lacks
consistency. If you ask the same question twice, the LLM might decide to search different
keywords or call entirely different tools. The context changes, so the answer changes.

## Predictable Retrieval

Instead of ordering an LLM to iteratively guess keywords, classic Retrieval Augmented
Generation (RAG), which separates the retrieval from the generation, might works better
depending on your use case or constraints. It bridges the semantic gap, matching the
underlying meaning of a user's query rather than just literal keyword matches.

A common misconception in the industry is that implementing RAG requires standing up a
dedicated Vector Database. In reality, dedicated vector databases are only necessary when
scaling to tens of millions or billions of rows. For the vast majority of enterprise use
cases below that scale, a standard Vector Index attached to your existing relational
database (such as `pgvector` in PostgreSQL) is more than enough.

When a query comes in, the system performs a mathematical search (cosine similarity)
against the vector index to fetch the top-K chunks, inject them into the LLM prompt and
let the LLM generate the answer.

Retrieval becomes a standard database read. It is fast, cheap, and highly predictable. The
math guarantees that identical queries will retrieve the exact same chunks every single
time.

## Forcing the LLM to Behave

RAG is by no means completely deterministic. Because even with strict top-K filter, you
still inject those chunks into a probabilistic LLM. But we can make the system **highly
reproducible** by configuring the LLM itself:

- Set the temperature to 0 (to remove creativity)
- Fix the LLM seed (for local models)
- Freeze the vector index

When you combine deterministic mathematical retrieval with strict generation parameters,
the entire system becomes reproducible. You can audit exactly why the AI generated a
specific response and trace it back to the exact source text where the data was ingested.

_(Note: Because GPUs calculate floating-point math in parallel, LLMs are never 100%
deterministic across different machines [^2], but for a single-node system,
`temperature=0` and a fixed seed are as close as we can get.)_

## Conclusion

AI models will never be perfectly deterministic, but they will get more accurate as
technology advances.

The rate of error might eventually drop so low that general consumers won't care. But for
critical sectors involving human life or strict compliance, that slight margin of error is
intolerable.

That is exactly why predictable retrieval systems like RAG, which (when scaled
appropriately with a vector index rather than an oversized vector database) prioritize
auditability and exact source citations over "agentic magic", will always have a place in
enterprise software.

## References

[^1]:
    Anthropic. _Model Context Protocol Specification_.
    https://modelcontextprotocol.io/specification

[^2]:
    PyTorch. _Reproducibility Documentation_.
    https://pytorch.org/docs/stable/notes/randomness.html
