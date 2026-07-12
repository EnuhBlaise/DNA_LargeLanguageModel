# Companion Notes — DNA Language Models Explained

Plain-language explanations of every concept in the tutorial. Read alongside `dna_language_model_tutorial.ipynb`, or on its own as a primer. No jargon is used without defining it.

---

## The big picture

We train a program to play one game: **read some DNA, predict the next letter.** Do that billions of times on real genomes and the program is forced to internalize the statistical "grammar" of DNA — which is what makes it useful for biology afterwards.

![pipeline](diagrams/01_pipeline.png)

Every large language model (ChatGPT included) is trained on this same next-token game. We are doing it on a 4-letter alphabet (`A C G T`) instead of English words.

---

## 1. DNA, in the amount you need

- DNA is a long chain built from four **bases**: adenine (`A`), cytosine (`C`), guanine (`G`), thymine (`T`).
- A **genome** is an organism's complete DNA. Sizes: human mitochondrial genome = 16,569 bases; *E. coli* ≈ 4.6 million; human = 3.2 billion.
- **bp** = *base pairs* = the length of a sequence in letters.
- **GC content** = percentage of bases that are `G` or `C`. It's a simple but biologically meaningful summary (affects how tightly DNA binds and where genes cluster).
- **Soft-masking**: real genome files write repetitive regions in lowercase (`acgt`). We uppercase everything so the model sees one consistent alphabet. (You saw exactly one stray lowercase base in the human MT genome.)
- DNA has real structure the model can learn: **codons** (3-base "words" that code for amino acids), **motifs** (short recurring regulatory patterns), genes, and repeats. This structure is why a language model works at all here — random letters would be unlearnable.

---

## 2. Tokenization — text to numbers

Neural networks only do arithmetic, so every letter must become a number (a **token ID**).

![tokenization](diagrams/02_tokenization.png)

- **Character-level** (what we use): one base = one token. Vocabulary size 4. Simple, lossless, and what modern models like HyenaDNA and Evo use.
- **k-mer**: group `k` consecutive bases into one token (e.g. `k=3` → `ATG`, `TGC`, …). Vocabulary size `4^k`. Sequences become shorter and each token already carries local context. **DNABERT** uses this.
- **Vocabulary**: the fixed set of possible tokens. **encode** maps letters → IDs; **decode** maps back.

---

## 3. The learning task and the context window

The model predicts the **next** base from the bases before it. We process fixed-length chunks called the **context window** (`block_size`). The correct answer at each position is just the next base, so the **target** is the input shifted left by one.

![next token](diagrams/03_next_token.png)

- **Batch**: several windows processed together for speed (`batch_size`).
- **Train / validation split**: we hold out ~10% of the data the model never trains on, so we can measure honest performance on *unseen* DNA rather than memorization.

---

## 4. The model: a Transformer, piece by piece

A **Transformer** is the neural network architecture behind essentially all modern language models. Ours is small enough to read in full.

**Embedding** — each token ID becomes a vector of learnable numbers (`n_embd` of them), the model's internal "meaning" for that base. A separate **positional embedding** encodes *where* in the window each base sits, because order matters.

**Self-attention** — the heart of the Transformer. For each position, the model computes how relevant every earlier position is, then mixes their information accordingly. This is how it captures dependencies like "this base pattern mirrors one seen 12 positions ago." A **head** is one such attention mechanism; **multi-head** runs several in parallel to capture different pattern types. Mechanically each head builds three vectors per token — **query** (what am I looking for), **key** (what do I offer), **value** (what I'll pass on) — and matches queries against keys.

**Causal mask** — since we predict the *future*, each position may only attend to itself and *earlier* positions, never later ones. A triangular mask enforces this. Without it the model could cheat by peeking at the answer.

![causal mask](diagrams/05_causal_mask.png)

**Transformer block** — attention followed by a small per-position **feed-forward network**, each wrapped with:
- a **residual connection** (add the block's input back to its output) — lets gradients flow and makes deep stacks trainable;
- **LayerNorm** — rescales activations to keep the numbers stable during training.

![block](diagrams/04_block.png)

We stack `n_layer` of these blocks, then a final linear **head** turns the top vectors into a score for each of the 4 possible next bases.

---

## 5. Training — how learning actually happens

- **Logits** — the model's raw scores for each possible next base. **Softmax** turns them into probabilities that sum to 1.
- **Loss (cross-entropy)** — one number measuring how much probability the model put on the *wrong* bases. Low loss = confident and correct.
- **Backpropagation** — the calculus that computes, for every one of the model's parameters, which direction would reduce the loss.
- **Optimizer (AdamW)** — takes a small step in that direction. **Learning rate** controls step size.
- **Iteration / step** — one batch processed and one weight update. We do `max_iters` of them.
- **Dropout** — randomly zeroing a fraction of activations during training so the model can't over-rely on any one path (reduces **overfitting**, i.e. memorizing the training data instead of learning general patterns).

You watch training work by watching the loss curve fall:

![training curve](diagrams/06_training_curve.png)

---

## 6. Evaluation — is it any good?

- **Perplexity = exp(loss)**. Interpretation: the effective number of bases the model is choosing between. Lower is better.
- **Baselines make the number meaningful:**
  - *Uniform* (know nothing) → perplexity **4.0**.
  - *Base-composition* (know only each letter's frequency) → ~**3.87** here.
  - Our trained model reaches ~**3.7**, beating both — proof it learned real dependencies between bases, not just how often each letter appears.
- Always report perplexity on the **validation** set (unseen data), never the training set.

---

## 7. Generation and probing what was learned

- **Autoregressive generation**: sample a next base from the model's probabilities, append it, feed it back, repeat. This produces brand-new DNA.
- **Sanity check via k-mers**: compare 3-base-word frequencies of real vs. generated DNA. High correlation (~0.7–0.8 for our tiny model) means it reproduced real local structure.

![kmer comparison](diagrams/07_kmer_compare.png)

- Our small char-level model captures **local** structure well but may drift on **global** composition (e.g. GC%). That's expected — more parameters, more data, and more training tighten it.

---

## 8. From toy to real research

| Our tutorial | Real research models |
|---|---|
| char-level, ~0.3–1M params | subword/k-mer or char, 10M–7B+ params |
| 1 Mbp of one genome | thousands of genomes, billions of bases |
| next-base (causal) objective | causal *or* masked-base (BERT-style) objective |
| generate DNA, measure perplexity | predict pathogenic mutations, find genes/promoters, classify sequences |

Real models to look up once you understand the above: **DNABERT / DNABERT-2**, **Nucleotide Transformer**, **HyenaDNA** (very long context), **GENA-LM**, **Evo**. Benchmarks to fine-tune on: **Genomic Benchmarks**, **GUE**.

**Masked vs. causal, in one line:** *causal* (ours) predicts the next base and is good for generation; *masked* (DNABERT) hides bases and predicts them from both sides, which is often better for classification tasks.

---

## Glossary quick-reference

- **base / nucleotide** — one letter of DNA (`A/C/G/T`).
- **bp** — base pairs; sequence length.
- **token** — smallest unit the model reads (here: one base).
- **vocabulary** — set of possible tokens.
- **embedding** — learnable vector representing a token or position.
- **context window / block_size** — how many bases the model sees at once.
- **batch** — several windows processed together.
- **attention** — mechanism for weighing the relevance of other positions.
- **head** — one attention mechanism; multi-head = several in parallel.
- **causal mask** — blocks attention to future positions.
- **residual connection** — adds a layer's input to its output.
- **LayerNorm** — normalization that stabilizes training.
- **logits** — raw output scores; **softmax** converts them to probabilities.
- **loss (cross-entropy)** — how wrong the predictions are.
- **backpropagation** — computes how to adjust weights.
- **optimizer / learning rate** — applies and scales the adjustment.
- **epoch / iteration** — a pass / a single update step.
- **overfitting** — memorizing training data instead of generalizing.
- **dropout** — regularization that randomly drops activations.
- **perplexity** — exp(loss); effective number of choices.
- **autoregressive generation** — sampling one token at a time, feeding it back.
- **k-mer** — a length-`k` substring; useful for tokenization and analysis.
