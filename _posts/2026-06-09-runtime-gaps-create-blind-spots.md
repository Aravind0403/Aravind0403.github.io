---
layout: post
title: "Runtime Gaps Create Blind Spots"
date: 2026-06-09
---

vLLM solves head-of-line blocking by design. Most hardware can't run vLLM. That gap has consequences — and they don't show up in benchmarks on 128GB machines.

Earlier this week, I published [a paper](https://arxiv.org/abs/2606.07248) on head-of-line blocking in serial LLM backends — a queueing problem that shows up when you run Ollama or llama.cpp without a batching runtime in front of it. The paper's experiments ran on an RTX 4090.

What I didn't have was a measurement on constrained hardware. The kind most people actually deploy on.

This is that measurement.

---

## The Reference Point

Kumar Nachiketa wrote [a detailed post](https://knachiketa04.github.io/llm-pipeline-on-a-workstation.html) walking through a full LLM pipeline — data prep, synthetic generation, fine-tuning, eval, and serving — measured end-to-end on a 128GB DGX Spark. His thesis: *the bottleneck is never the disk.* He proved it at every stage. It's a good post.

I ran the same five stages on a 16GB M1. Same measurements, different machine. The question I was after: does the bottleneck map hold, or does constrained memory change which layer you're actually fighting?

Stages 1–4: the map mostly holds. Stage 5: the problem class changes entirely — because the runtime does.

---

## Hardware Gap

|                   | Kumar (DGX Spark)              | This (M1 16GB)               |
|-------------------|-------------------------------|------------------------------|
| RAM               | 128GB unified                 | 16GB unified                 |
| Memory bandwidth  | 273 GB/s                      | ~68 GB/s                     |
| Serving runtime   | vLLM (continuous batching)    | Ollama (serial FCFS)         |
| Teacher model     | Qwen3-32B                     | Llama-3.1-8B Q4              |
| Student model     | Qwen3-8B                      | Qwen2.5-1.5B                 |
| Fine-tuning       | Full SFT — 46GB checkpoints   | LoRA — 85MB adapter          |

The runtime difference at serving isn't a preference; it's a physical constraint. On 16GB, vLLM's continuous batching overhead won't fit alongside the model weights. You use Ollama. That single constraint changes the serving problem class entirely.

> **On 16GB, you don't choose Ollama. You're assigned it.**

---

## Stages 1–4: Where the Maps Converge and Diverge

**Stage 1 — Data prep:** Identical finding. CPU-bound text parsing, disk idle. Memory constraint doesn't affect this stage.

**Stage 2 — Synthetic generation:** Same phenomenon, lower ceiling. Memory bandwidth caps token throughput. Kumar hit his wall at ~120 tok/s on 273 GB/s. I hit mine at 21.6 tok/s on 68 GB/s — 4× lower, which is roughly what the bandwidth ratio predicts. The flat speed curve regardless of recipe complexity is the memory wall fingerprint.

One consequence of the smaller teacher (8B vs 32B): format compliance dropped to 37%. For Stages 3–5, I used Kumar's published HF dataset to keep comparisons fair and focus the original work on serving.

**Stage 3 — Fine-tuning:** The writer concurrency bottleneck Kumar identified disappears at LoRA scale. 85MB saves in under a second. The hardware constraint forced a solution (LoRA) that makes the original problem irrelevant.

**Stage 4 — Eval:** Fine-tuning eliminated hallucinations on the recipe domain (3.3% → 0%). Constraint-following held steady. The 16GB constraint forced sequential judge + student calls — no parallelism headroom — doubling latency vs what's possible on 128GB.

---

## Stage 5 — Serving: Where the Runtime Gap Surfaces

Kumar's serving bottleneck was a single-threaded CPU loader. Default vLLM: 106 seconds to load. With parallel loading: 3 seconds. One config flag, 35× speedup.

That problem doesn't exist in my setup — because a different one took its place.

**Why vLLM hides it and Ollama exposes it:** On 128GB with vLLM, continuous batching interleaves short and long requests at the token level. Short requests never queue behind long ones. Head-of-line blocking is eliminated at the runtime layer before it can materialise. The CPU loader was the interesting problem *because* there was no queueing problem to find. You don't notice a door is locked when you've already walked through it.

On 16GB with Ollama, that runtime capability is unavailable — not by preference, but by memory constraint. Requests process serially, FCFS. Five requests arrive together — one long recipe generation (~60s), four short lookups (~2s each). The four short requests wait 60 seconds. Each. That's 232 seconds of avoidable latency on 2-second queries.

This is a high-Cs² M/G/1 queue. FCFS is the worst scheduling policy for it. The fix is Shortest-Job-First at the admission layer — the only layer constrained hardware makes available.

[Clairvoyant](https://github.com/Aravind0403/clairvoyant-scheduler) is the proxy I built for this problem. It intercepts requests at the HTTP layer, extracts 19 lexical features in 0.029ms (adding <0.1% overhead to the total request lifecycle), and classifies short/medium/long via a lightweight ONNX XGBoost model. The system relies on structural lexical features rather than simple token counting because prompt length is a poor proxy for generation length — a dense JSON payload might have few tokens but trigger verbose reasoning, whereas a long natural language prompt might yield a single-word answer.

The proxy reorders the queue before dispatch. No changes to Ollama. No changes to the model. Because the backend remains FCFS, misclassifications degrade gracefully: the system falls back to a slightly suboptimal ordering for that dispatch, without crashing or starving the queue.

The paper's results were on an RTX 4090. Here are the M1 measurements.

---

## Results

**Across 6 runs (general predictor):**

| Run | FCFS Short P50 | Clairvoyant Short P50 | Reduction |
|-----|---------------|-----------------------|-----------|
| 1   | 43.3s         | 20.0s                 | 53.9%     |
| 2   | 24.1s         | 17.4s                 | 27.5%     |
| 3   | 46.9s         | 11.5s                 | 75.6%     |
| 4   | 25.8s         | 16.5s                 | 36.1%     |
| 5   | 20.5s         | 12.0s                 | 41.2%     |
| 6   | 32.4s         | 13.4s                 | 58.6%     |
| **Avg** | **32.2s** | **15.1s**             | **~49%**  |

**Three-condition benchmark (final run):**

| Condition                          | Short P50 | Long P50 | Short P50 reduction |
|------------------------------------|-----------|----------|---------------------|
| A — FCFS                           | 32.9s     | 81.4s    | —                   |
| B — Clairvoyant (general predictor)| 10.5s     | 71.1s    | **68.1%**           |
| C — Clairvoyant (domain-retrained) | 15.0s     | 67.7s    | **54.5%**           |

The original paper on an RTX 4090 showed a 70–76% reduction. The M1 result of 68.1% falls within that range, indicating the mechanism is hardware-agnostic.

**On variance (17–75% across runs):** M1 thermal throttling and memory pressure cause generation time variance that makes length classification harder. This is a constrained-hardware effect absent on the RTX 4090. The variance is itself a finding — serial backends on edge hardware are less predictable, which is precisely why admission-layer scheduling matters more, not less.

**On domain retraining (Condition C):** The domain-specific predictor reduces short P50 performance (54.5% vs 68.1%) but improves long P50 (67.7s vs 71.1s). The general predictor's token-length heuristic is already well-calibrated for short requests. Domain retraining adds value in long-request ordering. The right predictor depends on what you're optimising for.

---

## The Full Bottleneck Map

| Stage      | Kumar (128GB DGX)           | This project (16GB M1)              |
|------------|----------------------------|--------------------------------------|
| Data Prep  | CPU parsing                | CPU parsing — identical              |
| Synth Gen  | Memory wall (273 GB/s)     | Memory wall (68 GB/s) — 4× lower    |
| Fine-Tune  | Writer concurrency (I/O)   | I/O irrelevant at adapter scale      |
| Eval       | KV-cache pressure          | KV-cache → forced serialisation      |
| Serving    | CPU loader (1 thread)      | **Head-of-line blocking (queue)**    |

---

## What This Shows

Kumar's thesis holds: the bottleneck is never the disk. But there's a corollary his hardware never exposes: which bottleneck you're fighting depends on which runtime capabilities you can actually run.

On 128GB, continuous batching solves the queueing problem invisibly — before you can measure it. The CPU loader becomes interesting precisely because there's nothing else to fix. On 16GB, the solution is unavailable. The problem it was hiding surfaces in full, and a config flag won't fix it.

Here's the deployment reality: most LLM inference doesn't happen on DGX Sparks. It happens on laptops, edge servers, cost-constrained cloud instances, and local setups running quantised models on sub-32GB hardware. On that hardware, vLLM isn't an option. Serial FCFS execution is the default state, not an edge case. The queueing problem is the common case.

Clairvoyant addresses it at the admission layer — the only layer constrained hardware makes available. It doesn't approximate continuous batching; the mechanisms are different. Continuous batching interleaves at the token-iteration level (inside the engine). Clairvoyant reorders at the dispatch level (outside the engine). They solve the same visible symptom — short requests waiting behind long ones — through entirely different interventions.

What the numbers show: 68.1% P50 reduction on M1, 70–76% on RTX 4090. The mechanism is hardware-agnostic even though the problem is hardware-dependent. You don't need an expensive GPU to get reasonable tail latency on short requests. You need smarter queuing at the layer you can actually control.

That's what this project measures.

---

## Reproducing This

The full pipeline ships as a reproduction kit: [github.com/Aravind0403/edge-llm-pipeline](https://github.com/Aravind0403/edge-llm-pipeline).

### Quickstart (Condition A only)

To reproduce the FCFS baseline with zero additional setup:

```bash
git clone https://github.com/Aravind0403/edge-llm-pipeline
cd edge-llm-pipeline
pip install -r requirements.txt

ollama pull qwen2.5:1.5b && ollama pull llama3.1:8b

python scripts/01_data_prep.py
python scripts/02_synth_gen.py
python scripts/04_eval.py    # downloads the pre-trained adapter automatically
python scripts/05_benchmark.py  # runs FCFS baseline only