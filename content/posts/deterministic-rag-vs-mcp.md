## RAG vs MCP

"MCP is the future, it will revolutionize how AI interacts with APIs", they say.

Before casting stones, let's see how classic RAG flow works. First, the user ingest data.
Then the system embeds the text into a vector database. When query comes in, the system
performs a mathematical search(cosine similarity) to fetch the top-K chunks, inject them
into LLM prompt and let the LLM generates the answer.

Now let's see how different MCP flow for the same use case. Instead of a vector database,
LLM is given a set of tools (e.g., `read_file`, `search_database`). The user writes a
prompt, and the LLM decides which tools to call, reads the output, and generates the
answer.

On paper, MCP sounds more advanced. It is way more flexible, indeed. But in practice, it
is much harder to build, secure, and test. Infinite loops, latency, and the fact that LLM
might hallucinate the tool arguments are some of the things that need to be handled.

But the biggest difference is **consistency**.

With MCP, if you ask the same question twice, the LLM might decide to call different
tools, or search for different keywords. The context changes, so the answer changes.

With RAG, the retrieval is strictly mathematical. By enforcing a strict top-K filter, the
same question will always retrieves the exact same chunks.

RAG is by no means deterministic. Because even with strict top-K filter, you still inject
the chunks to a prompt and pass it to LLM. But we can make it **more** deterministic by
configuring the LLM itself:

- set the temperature to 0
- fix the LLM seed (for local models)
- freeze the vector index

When you combine deterministic retrieval with better generation parameter, the entire
system becomes reproducible. You can audit exactly why the AI said what it said and trace
it back to the source where you or user ingests the data

_Note: Because GPUs calculate floating point math in parallel, LLMs are never 100%
deterministic across different machines, but for a single-node system, `temperature=0` and
a fixed seed are as close as we can get_

AI models will never be deterministic, but it can get more accurate as everything advance
but it will never be 100% accurate and free of error. It's just that the rate of error
might be so low people don't notice it or won't be bothered that makes people think AI is
gonna replace everything, and it's fine that most use case maybe fine with that but
critical sector that involves humans life, safety, etc might not tolerate that slight
margin of error.
