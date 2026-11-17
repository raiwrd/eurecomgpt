# Phase 3 — Comparison Writeup

Copy this file to `submission/phase3_comparison.md`, fill in every answer slot, and commit it:

```bash
cp phase-3-mapreduce-spark/comparison_template.md submission/phase3_comparison.md
```

**How to fill it in.** Each answer sits between a pair of `<!--answer:...-->` markers. Replace
the `TODO` line with your answer — **leave the markers themselves untouched**, they are how the
grader finds your answers. Anything you write outside the markers is ignored, so add extra
prose, tables or images freely.

Numbers in section 1 must **match `submission/phase3_report.json`** — copy them across, do not
retype from memory; they are checked against it.

---

## 1. Facts from your runs

Numbers only (e.g. `12`, `20.3`, `0.041`).

| | Value |
|---|---|
| Map tasks in the cloud job (`mapreduce.num_splits`) | <!--answer:num_map_tasks-->12<!--/answer--> |
| Reduce tasks in the cloud job (`mapreduce.num_reducers`) | <!--answer:num_reduce_tasks-->4<!--/answer--> |
| Cloud MapReduce wall time, seconds (`mapreduce.cloud_elapsed_s`) | <!--answer:cloud_elapsed_s-->48.4<!--/answer--> |
| Local single-process wall time, seconds (`mapreduce.local_elapsed_s`) | <!--answer:local_elapsed_s-->0.012<!--/answer--> |
| Spark wall time for the `collect`, seconds (`tfidf.spark_elapsed_s`) | <!--answer:spark_elapsed_s-->4.1<!--/answer--> |
| Number of **stages** in the Spark job (read off notebook 2b / 3a) | <!--answer:num_stages-->4<!--/answer--> |
| Task **retries** you counted in your `--chaos` run (0 if none happened) | <!--answer:chaos_retries-->7<!--/answer--> |

**Explain the timings.** The three wall times above cover the same 60 documents, but not the
same job: the single-process run and the cloud MapReduce are *the same word count*; Spark's
time is for *TF-IDF*, which is more work (two `reduceByKey`s and a `join`). Rank the three and
explain what each is spending its time on. Say which comparison is like-for-like, and what the
Spark number — a heavier job, on one machine — tells you about where the cloud job's time goes.
(~80 words)

<!--answer:timing_explanation-->
Fastest to slowest: local word count (0.012 s), Spark TF-IDF (4.1 s), cloud MapReduce (48.4 s). The like-for-like pair is local vs cloud: same word count, same result, but the cloud is about 4000x slower. The local run only does the computation in memory. Spark does more work but still finishes in seconds; its time is mostly job/stage scheduling and Python worker start-up. The cloud job spends almost all its time on overhead, not computation: HTTP calls to Cloud Run (with cold starts), reading and writing about 65 JSON files on Cloud Storage, workflow scheduling, retry back-off and the barrier. Since even a heavier job on one machine takes only 4 s, the 48 s is coordination and I/O cost on a corpus far too small to pay it back.
<!--/answer-->

**What the counts tell you.** From your stage count: how many shuffles did the Spark job have,
and which operations in `spark_tfidf.py` caused them? From your retry count: roughly what
fraction of the 16 task attempts was retried, and is that what `--chaos 0.3` should produce?
(~50 words)

<!--answer:counts_interpretation-->
4 stages means 3 shuffles: the `reduceByKey` in `term_freq`, the `reduceByKey` in `doc_freq`, and the `join` in `build_tfidf_rdd`. In the chaos run 7 attempts failed for 16 tasks, i.e. 23 attempts in total, so about 7/23 = 30% of attempts failed (about 0.44 retries per task). This matches `--chaos 0.3`: the expected number of failed attempts is 16 x 0.3/0.7 = 6.9.
<!--/answer-->

---

## 2. Spark: transformations, actions, lineage

**Transformations vs actions.** Go through `spark_tfidf.py` and the notebook's Spark cells.
Name each RDD operation and say whether it is a transformation or an action. What happened
when you ran `build_tfidf_rdd` — and what happened when you ran `collect`? (~60 words)

<!--answer:transformations_vs_actions-->
Transformations (lazy): `parallelize` (creates the source RDD), `flatMap` and `reduceByKey` in `term_freq`, `map` and `reduceByKey` in `doc_freq`, then `map` (tf_by_term), `join` and the final `map(to_row)` in `build_tfidf_rdd`. The only action is `collect`. Running `build_tfidf_rdd` returned instantly and computed nothing: it only recorded the lineage graph, which `toDebugString` printed. `collect` triggered one job: Spark cut the lineage into 4 stages (12 tasks), ran them and returned 5975 rows to the driver in 4.1 s.
<!--/answer-->

**Lineage and recovery.** Using the lineage you printed in 2b and the timeline in 3a: if the
worker holding one output partition of the *first* `reduceByKey` died, what would Spark have
to recompute, and what could it reuse? Name the stage or shuffle boundary where the recovery
starts. (~50 words)

<!--answer:lineage_recovery-->
Recovery starts at the first shuffle boundary, `ShuffledRDD[3]` (the TF `reduceByKey`, between stage 0 and stages 1/2). Spark recomputes only the lost partition: it re-fetches that partition's data from the stage-0 shuffle files and re-runs the tasks that consumed it. Only if those map-side shuffle files were also lost would it re-run the stage-0 tasks from `parallelize`. The other partitions and the DF shuffle files are reused, not recomputed.
<!--/answer-->

---

## 3. Your cloud MapReduce, mapped onto Hadoop

Name the GCP service or file that played each role (one line each):

- **JobTracker** (schedules tasks, waits for the map phase, retries failures):
  <!--answer:role_jobtracker-->Cloud Workflows: the `mr-wordcount` workflow defined in `workflow.yaml`<!--/answer-->
- **Workers** (run your map and reduce code):
  <!--answer:role_workers-->The Cloud Run function `mr-worker` (`main.py`), one invocation per map or reduce tas<!--/answer-->
- **HDFS** (holds inputs, shuffle files, outputs):
  <!--answer:role_hdfs-->Cloud Storage: the bucket `eurecomgpt-512008-eurecomgpt` (input splits, shuffle part files, output partitions)<!--/answer-->
- **Partitioner** (decides which reducer owns a key):
  <!--answer:role_partitioner-->`partition()` in `mapreduce.py`: `zlib.crc32(word) % num_reducers`, called by `run_map` in `main.py`<!--/answer-->

**The barrier.** There is no explicit "wait for all mappers" step in `workflow.yaml`. Where,
exactly, is the barrier between the map and reduce phases — which construct provides it?
(~30 words)

<!--answer:barrier_location-->
The barrier is the end of the `map_phase` step. It is a `parallel` step with a `for` loop, and a `parallel` step only completes when all its branches have finished. So `reduce_phase`, the next step, cannot start before the last map task (including its retries) is done.
<!--/answer-->

**Idempotence.** The workflow retries a task that failed. Why is that safe with the worker as
written, and what would go wrong if `run_map` *appended* its pairs to the shuffle file instead
of overwriting it? (~60 words)

<!--answer:idempotence-->
A map task reads a fixed input split and writes its output to fixed file names (`split-NNN/part-r.json`), overwriting them. Running it twice gives exactly the same files as running it once, so a retry cannot change the result. Also, the chaos crash happens before any work. If `run_map` appended instead, a task that crashed after a partial write, or simply ran twice, would leave duplicate pairs in the shuffle files, and the reducers would count those words twice, giving wrong totals.
<!--/answer-->

**The chaos run.** What did you see in the execution log and in Logs Explorer when you ran
with `--chaos 0.3`? Did the job still succeed, and was the final answer still identical to the
local reference? (~50 words)

<!--answer:chaos_observations-->
With `--chaos 0.3`, 7 task attempts were rejected with HTTP 503 (the simulated worker crash); I counted them in Cloud Logging as `mr-worker` requests with status 503. The workflow's `try`/`retry` re-invoked each failed task with exponential back-off, and every task eventually succeeded. The execution ended SUCCEEDED in 48.4 s, with 2221 distinct words and `identical: True` (matches_local true), so the result was the same as the local reference.
<!--/answer-->
