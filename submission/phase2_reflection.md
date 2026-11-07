# Phase 2 — Reflection: where does parallelism stop helping?

Copy this file to `submission/phase2_reflection.md`, fill in every answer slot, and commit it:

```bash
cp phase-2-tiny-gpt/reflection_template.md submission/phase2_reflection.md
```

**How to fill it in.** Each answer sits between a pair of `<!--answer:...-->` markers. Replace
the `TODO` line with your answer — **leave the markers themselves untouched**, they are how the
grader finds your answers. Anything you write outside the markers is ignored, so add extra
prose, tables or your threading plot freely.

Numbers in section 1 must **match `submission/phase2_report.json`** — copy them across, do not
retype from memory; they are checked against it.

---

## 1. Your measurements

| | Value |
|---|---|
| Naive attention, ms (`attention.naive_ms`) | <!--answer:attn_naive_ms-->249.924<!--/answer--> |
| Vectorized attention, ms (`attention.vectorized_ms`) | <!--answer:attn_vectorized_ms-->1.097<!--/answer--> |
| CPU training, ms for 100 steps (`devices.cpu_ms`) | <!--answer:cpu_ms-->435848.2<!--/answer--> |
| GPU training, ms for 100 steps (`devices.gpu_ms`) | <!--answer:gpu_ms-->13787.6<!--/answer--> |
| Final training loss (`training.final_loss`) | <!--answer:final_loss-->1.4666<!--/answer--> |

**The thread sweep.** Give your four timings from `threads` — 1, 2, 4 and 8 threads — and say
which count was fastest. (verbatim: the numbers, in order)

<!--answer:thread_sweep-->
1 thread: 2212.303 ms, 2 threads: 2403.856 ms, 4 threads: 2232.667 ms, 8 threads: 2396.835 ms. Fastest: 1 thread (2212.303 ms).
<!--/answer-->

---

## 2. Intra-node parallelism, three ways

**Why vectorizing wins.** Both attention implementations do the same arithmetic and produce the
same result (the unit test checks that). Explain where the naive version's time actually goes,
and what the vectorized one does differently — in terms of the machine, not the Python.
(~70 words)

<!--answer:vectorization_why-->
The naive version spends its time in the interpreter: every multiply-add of QKᵀ and of weights·V is a separate scalar operation with bytecode dispatch, type checks and a loop jump, and it reuses the cache poorly. The vectorized version makes one call into compiled BLAS, which streams contiguous blocks through the cache and uses AVX SIMD registers to do 8–16 floating-point operations per instruction. Same arithmetic, far fewer instructions: 249.924 ms → 1.097 ms (227.8×).
<!--/answer-->

**Where the thread curve flattens, and why.** Using your sweep: more threads stopped helping —
or started hurting — at some point. Say where, and give the reason. Your Colab runtime's vCPU
count is part of the answer; so is what the threads have to coordinate on. (~80 words)

<!--answer:threading_plateau-->
The curve is flat from the start: 1 thread was fastest (2212.303 ms); 2 threads were 8.7% slower (2403.856 ms), 4 threads about equal (2232.667 ms) and 8 threads 8.3% slower (2396.835 ms). The Colab runtime has 2 vCPUs, i.e. one physical core with two hyper-threads sharing the same execution units and AVX unit, so extra threads add almost no compute. They must also split every matmul, synchronise at the end of each operator and share caches and memory bandwidth; on this small model that coordination costs more than it saves.
<!--/answer-->

**What made the GPU faster.** The model, the data and the step count were identical; only the
device changed. Say what about a transformer training step suits a GPU, and name one workload
that would *not* see this speed-up. (~70 words)

<!--answer:gpu_why-->
A training step is dominated by large, independent matrix multiplications (QKV projections, QKᵀ, the MLP) in both the forward and backward pass. The T4 runs them on thousands of lock-step SIMT threads (2,560 CUDA cores plus tensor cores) with much higher memory bandwidth, so 100 steps went from 435848.2 ms to 13787.6 ms (31.6×). Autoregressive text generation would see little of this speed-up: it produces one byte at a time, each step depends on the previous one, and its tiny kernels are dominated by launch overhead.
<!--/answer-->

---

## 3. The question

**Where does parallelism stop helping, and why?** Pull the three experiments together. Your
answer should distinguish the *kinds* of limit you met — work that cannot be divided, hardware
you do not have, coordination that costs more than it saves, data that has to move — and say
which one bound each experiment. One concrete number from section 1 per claim. (~150 words)

<!--answer:parallelism_limit-->
Each experiment hit a different limit. Vectorization was not limited by the hardware but by interpreter overhead: once the work went to BLAS, SIMD did the same arithmetic in 1.097 ms instead of 249.924 ms. The thread sweep hit hardware I did not have plus coordination cost: with 2 vCPUs (one physical core), going past 1 thread gave no gain and up to 8.7% loss (2403.856 vs 2212.303 ms), because splitting and synchronising each operation cost more than it saved. The GPU gave 31.6× (435848.2 → 13787.6 ms) but no more, because part of every step cannot be divided or must move data: Python control flow, batch sampling on the CPU, host-to-device copies and kernel launches stay serial (Amdahl's law), and the small 4.9M-parameter model does not keep the GPU fully busy. All runs computed the same thing: training still converged to a final loss of 1.4666.
<!--/answer-->

**What you would measure next.** One experiment this phase did not run that would sharpen the
answer, and what result would change your mind. (~50 words)

<!--answer:next_experiment-->
Repeat the CPU-vs-GPU comparison with 4× and 16× larger batches and a wider model. If the GPU speed-up grew well beyond 31.6× while CPU time grew linearly, the limit was launch and transfer overhead, not compute; if it stayed near 31.6×, the GPU itself was saturated and my explanation would be wrong.
<!--/answer-->