# LLM Context Mechanism Standard

This standard records why large context must be engineered rather than simply
filled. It explains the model-level mechanisms that make a focused context
outperform a larger unfocused one. It is the mechanism companion to PROMPT.md,
which defines how to write prompts that exploit these mechanisms.

## Background: Attention Is Finite And Uneven

Self-attention compares every token with every other token, so cost grows as
O(N^2). 10k tokens is about 100 million attention scores, 100k is about 10
billion, 1M is about 1 trillion. Long-context models reduce this with sparse,
block, or sliding-window attention, but the reduction is an approximation. It is
not a guarantee that every token is used well.

Even with unlimited compute, attention is diluted. Every token competes for the
same attention budget, and the model has no persistent symbolic note saying
"this is the important part." Instructions that are technically still in context
can lose to nearer or stronger signals. This is why models appear to "forget"
rules that are still present.

Two consequences follow directly. They set the priority for context design.

## 3. Position Determines Effective Attention

Models attend best to information that is:

- near the beginning of the context,
- near the end of the context,
- or repeatedly reinforced.

Information buried in the middle receives less effective attention. This is the
"lost in the middle" effect. Increasing the context window does not make the
model use every token equally well.

Design rule:

- Place the highest-authority content (authority order, output contract, current
  task) at the edges of the context, not in the interior.
- Reinforce the current task rather than stating it once deep in the middle.
- Do not let critical evidence sink into a long undifferentiated interior.
- Treat position as part of the contract. Where a rule sits changes how reliably
  it is applied.

See PROMPT.md for the prompt-level structures (authority order, scoped rules,
output contracts) that make this placement explicit.

## 4. Working Context Over Full History

Because attention is finite and position sensitive, robust systems do not append
everything forever. They maintain a working context and bring in only what the
current task needs:

```text
Conversation
  -> summarize old messages
  -> keep only active state
  -> retrieve relevant documents
  -> current task
  -> LLM
```

The model rarely sees the entire lifetime history verbatim. The context window
behaves like RAM. Most information stays in external storage and is loaded only
when the task needs it.

Design rule:

- Curate the working set for each call. Summarize or drop stale turns instead of
  carrying them verbatim.
- Retrieve evidence against a specific representation question, not a broad
  similarity dump. This mirrors the retrieval rule in PROMPT.md.
- Prefer a well-managed smaller context over a large context full of stale or
  irrelevant material. A focused 50k-token context routinely outperforms a
  poorly managed 500k-token one.
- The core challenge is not fitting everything in. It is deciding what belongs
  in working memory for this specific task.

This is why the discipline is called context engineering, not just prompt
engineering.

## Relationship To Prompt Design

Position (section 3) and working context (section 4) are the mechanism. The
prompt standard is how a single call exploits them. Explicit sections tell the
model what to pay attention to, and edge placement plus a curated working set
make sure that attention lands where it matters. See PROMPT.md, "Explicitness
Directs Attention."
