# 01 · From Transformer to "Generating One Token": a Five-Step Derivation

[中文](01-from-transformer-to-token.zh.md)

> No deep-learning prerequisites. All matrix sizes come from a real Qwen3.5-0.8B checkpoint (see [chapter 03](03-checkpoint-anatomy-qwen3-5-0.8b.md)) — not textbook assumptions.

## Step 1: Words become numbers (Embedding)

The model doesn't know letters, only numbers. First, a giant lookup — the **embedding matrix E** (248,320 rows × 1024 columns):

```
"I love apples" → tokenizer → token ids: [314, 8721, 19526, 8891] (illustrative)
lookup → 4 vectors of 1024 dims
```

This table is itself part of the weights (~254M numbers). At this point the vectors are inert: "apple" and "keynote" know nothing of each other.

## Step 2: Attention — letting words see each other

Problem: "apple" doesn't know whether it sits in a fruit stand or a product launch. Attention gives each word three role cards (each obtained by multiplying a projection table):

- **Q** (query — what am I looking for)
- **K** (key — how can I be found)
- **V** (value — the information I actually carry)

Attention of word i toward word j = `q_i · k_j` (dot product = compatibility), softmax-normalized; output = the V's mixed by those weights.

**Hand-computed mini example** (2-D toy, identity projection tables):

```
word1 "apple" x₁=[1,0]   word2 "launch" x₂=[0,1]

word1 scores: self q₁·k₁ = 1; word2 q₁·k₂ = 0
softmax([1, 0]) ≈ [0.73, 0.27]
word1 new vector = 0.73·v₁ + 0.27·v₂ = [0.73, 0.27]
```

"apple"'s vector now contains part of "launch" — that is context fusion. GPT adds one house rule (causal mask): a word may only attend to itself and words to its left — no peeking at the future.

## Step 3: One full layer (real sizes)

```
Input X: L tokens × 1024 dims
① q/k/v projections (1024→4096/512/512) → attention scoring & mixing → o_proj back to 1024
② residual + RMSNorm (stabilizes numbers; only 1024 parameters)
③ MLP (SwiGLU): gate/up up-project 1024→3584, SiLU gate multiplies elementwise, down projects back to 1024
④ residual + RMSNorm
```

One layer = 7 large tables (q/k/v/o + gate/up/down); every number in every table participates in a multiplication for every token passing through.

## Step 4: GPT = N chained layers + one output table

```
final output = f₂₄( f₂₃( … f₂(x) … ) )     ← function composition, nested
```

**The mathematical root of "no layer can be skipped"**: layer 2's input is layer 1's output; to know what layer 24 computes you must compute layer 23, all the way back. Finally the LM head (this model reuses E transposed — "tied") projects 1024 dims to vocabulary size:

```
logits = Eᵀ · h ∈ R^248320  →  softmax  →  probability distribution of the next token
```

## Step 5: The autoregressive loop — the engine of everything

```python
tokens = tokenize(prompt)
while not done:
    logits = forward(tokens)          # through all layers × 7 tables each + LM head
    next_id = sample(softmax(logits[-1]))
    tokens.append(next_id)            # append, then do it all again
```

**Each iteration = one full forward pass = touching every single weight number.** Writing 100 tokens runs the pipeline 100 times. KV caching only avoids recomputing historical K/V — the weight tables are still read in full every pass. That is the origin of the 54GB bill in [chapter 02](02-memory-bound-and-batch.md).

## Three takeaways

1. A model has no answer database, only a processing pipeline — generation is computed, not looked up;
2. Layered function composition → every forward pass must traverse every table; nothing can be skipped;
3. Autoregression is inherently sequential (token N+1's input depends on token N) → the full-weight-read fixed cost repeats once per token.
