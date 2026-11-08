---
title: "The Tokenizer Mismatch in RAG Pipelines"
date: 2026-11-04
draft: false
description:
  "Why matching your text splitter's chunk size to your embedding model's context limit is
  a recipe for disaster."
ShowDescription: false
tags: ["AI", "RAG", "Java", "Spring AI"]
---

<p align="center">
  <img src="/InitialD.gif" alt="Tokenizer Drift" width="100%" />
  <i>Tokenizer Drift</i>
  <br>
</p>

When building a Retrieval Augmented Generation (RAG) pipeline, we usually focus on the
models. We pick the best local LLM, we pick the best embedding model, and we wire them
together in Spring AI.

But there is a hidden trap in the pipeline that is rarely talked about. It lives in the
step right before the embedding model, called **Chunking**.

To understand why your local RAG pipeline might be silently failing, you first have to
understand why your application is using two different tokenizers at the exact same time.

## How Spring AI Chunks and Embeds

A standard RAG pipeline has two distinct steps:

1. **Splitting:** You take a large document and cut it into smaller chunks.
2. **Embedding:** You take those chunks and turn them into numerical vectors.

In a Spring AI application, these two steps are handled by completely separate components.

**The Splitter**

You use `TokenTextSplitter` to cut the text. How does it know how big 500 tokens is?
Spring AI is designed to be provider-agnostic (it supports OpenAI, Anthropic, Ollama,
etc.). To count tokens generically, it uses a Java port of OpenAI's Byte Pair Encoding
(BPE) tokenizer (via `jtokkit`).

**The Embedding Model**

Once the chunk is created, Spring AI sends it to your local Ollama instance. The model
(`mxbai-embed-large`) receives the raw text and processes it. But to process it, the model
has to tokenize it _again_ using its own internal algorithm, which is a BERT-based
WordPiece tokenizer.

## The Token Limit Trap

`mxbai-embed-large` has a hard context limit of 512 tokens. If you send it a chunk larger
than that, it will not throw an error. As a benchmark by TigerData notes: _"It silently
truncates, so retrieval quality drops without any warning in your logs."_ [^1] The final
sentence of your chunk is simply deleted from the vector, and your RAG system becomes
blind to that information.

To prevent this, your logical instinct is to configure your Spring AI `TokenTextSplitter`
to exactly 512 tokens. But it won't work the way you expect.

## 512 Tokens Aren't Exactly 512 Tokens

Because Spring AI (BPE) and `mxbai` (WordPiece) use different vocabularies, they count the
same text differently.

- **Spring AI (BPE):** Uses a massive vocabulary (~100,000 tokens). It is trained to keep
  whole, common words together.
- **mxbai (WordPiece):** Uses a smaller vocabulary (~30,000 tokens). If a word isn't in
  its exact dictionary, it breaks it down into smaller subwords.

Let's use a word like `Dockerization` as an example.

- To Spring AI's tokenizer, this is likely 2 tokens: `[Docker][ization]`.
- To the embedding model, it has to split it further: `[Dock][##er][##ization]`. That is 3
  tokens.

If your chunk contains a lot of technical jargon, the embedding model will use more tokens
than your Java code did. The text itself doesn't change. Spring AI does not send "512
tokens" to the model, but rather it sends the raw string of text. The difference is how
the second tokenizer counts it. Let's say Spring AI measures your text as 512 tokens. It
sends that text to `mxbai`. Because `mxbai` uses a different dictionary, it chops
technical words into more pieces. When it finishes counting the exact same text, it could
count 550 tokens.

Because the model's hard limit is 512, those extra 40 tokens are silently truncated. The
last few sentences of your chunk are completely deleted.

To fix this, production systems apply a safety margin. **The Open Notebook** project
explicitly defaults its chunk size to **400 tokens** when using `mxbai-embed-large` [^2].
Their documentation states this is to stay safely below the 512 token ceiling after
accounting for tokenizer mismatch and splitter overshoot.

If you use a 400-token chunk, you have a safe buffer. Your Java code measures `400`
tokens. It sends the text to `mxbai`, and because of the tokenizer mismatch, `mxbai` might
count it as `430` tokens. That is still safely under the 512 limit. The model reads the
entire chunk, and the vector accurately represents the full text.

When you are configuring a RAG pipeline, the text splitter and the embedding model must be
treated as a coupled system. You can't just trust the default configurations. You have to
respect the hardware limits of the model and leave a buffer for the tokenizer mismatch.

## References

[^1]: TigerData. "Best Open Source Embedding Models for RAG in 2026."

[^2]: Open Notebook. "`OPEN_NOTEBOOK_CHUNK_SIZE` Defaults to 400 Tokens."
