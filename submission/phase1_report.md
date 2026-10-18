# Phase 1 — Comparison Report

Copy this file to `submission/phase1_report.md`, fill in every answer slot, and commit it:

```bash
cp phase-1-echo-bot/report_template.md submission/phase1_report.md
```

**How to fill it in.** Each answer sits between a pair of `<!--answer:...-->` markers. Replace
the `TODO` line with your answer — **leave the markers themselves untouched**, they are how the
grader finds your answers. Anything you write outside the markers is ignored, so add extra
prose, tables or images freely.

Numbers must **match `submission/phase1_report.json`** exactly — that file is produced by
`measure.py` and is what your figures are checked against. Copy them across; do not retype from
memory.

---

## 1. Setup facts

**Your image size**, in MB, from `gcloud artifacts docker images list` (number only, e.g. `142`):

<!--answer:image_size_mb-->
47.675
<!--/answer-->

**The custom VPC and subnet you created** (names only):

<!--answer:vpc_name-->
echo-net
<!--/answer-->

<!--answer:subnet_name-->
echo-subnet
<!--/answer-->

---

## 2. Measurements

Copy these from `submission/phase1_report.json`. Cold is `cold_ms`; warm median is
`stats.median`. Numbers only, in milliseconds (e.g. `812.4`).

| Platform | Cold (ms) | Warm median (ms) |
|---|---|---|
| IaaS VM | <!--answer:vm_cold_ms-->266.6<!--/answer--> | <!--answer:vm_warm_median_ms-->210.9<!--/answer--> |
| Cloud Run | <!--answer:cloudrun_cold_ms-->1761.4<!--/answer--> | <!--answer:cloudrun_warm_median_ms-->641.2<!--/answer--> |
| Cloud Function | <!--answer:function_cold_ms-->702.4<!--/answer--> | <!--answer:function_warm_median_ms-->639.0<!--/answer--> |

Include your histogram or plot of the warm distribution below (an image, or the ASCII chart
`measure.py` prints). This is read by a human, not the grader:

<!--answer:latency_plot-->
<pre>
Phase 1 — deployment measurements
------------------------------------------------------------
[OK  ] vm        cold=  266.6ms  warm_median=   210.9ms
            #############
[OK  ] cloudrun  cold= 1761.4ms  warm_median=   641.2ms
            ########################################
[OK  ] function  cold=  702.4ms  warm_median=   639.0ms
            ########################################
------------------------------------------------------------
</pre>
<!--/answer-->

---

## 3. Per-platform analysis

**Scaling behaviour.** Imagine requests arriving one at a time, then ten at once, then a
hundred. For each of the three platforms, answer concretely:

- Does anything get created automatically to absorb the load — and if so, *what* exactly?
  (One platform here creates nothing at all.)
- What is the hard limit, and where does it come from — a flag you passed yourself, or the
  size of the machine you picked?

(~80 words. Your `--machine-type=e2-micro` and `--max-instances=3` are both concrete answers
to the second question.)

<!--answer:scaling_behaviour-->
For the **IaaS VM**, nothing is created automatically; requests queue on a single instance limited strictly by the chosen `--machine-type=e2-micro` size. For **Cloud Run**, new container instances spin up automatically up to the hard limit defined by `--max-instances=3`. For **Cloud Functions**, the infrastructure scales out automatically to handle concurrency, capped by the function's concurrency limits or default platform ceilings, though in our setup it relies on platform defaults and configuration flags.
<!--/answer-->

**Deployment effort.** What did you have to build, configure and maintain to get each of the
three running? (~100 words)

<!--answer:deployment_effort-->
Deploying to the **IaaS VM** required building a Docker image, provisioning a custom VPC and subnet, configuring firewall rules, and manually managing the container runtime on a persistent virtual machine. For **Cloud Run**, effort was lower: we built and pushed the container image to Artifact Registry and let the platform manage serverless routing, scaling, and TLS termination. For **Cloud Functions**, effort was minimal regarding infrastructure maintenance, requiring only packaging the function code and deploying it directly to a managed serverless runtime environment.
<!--/answer-->

---

## 4. Why do two of your three numbers look alike?

With Cloud Run and your Function both deployed, run `gcloud run services list` and paste the
**complete output** here, unedited:

<!--answer:run_services_list-->
✔
SERVICE: echo
REGION: us-central1
URL: https://echo-1041791582068.us-central1.run.app
LAST DEPLOYED BY: service-1041791582068@gcf-admin-robot.iam.gserviceaccount.com
LAST DEPLOYED AT: 2026-10-18T18:31:07.647809Z

✔
SERVICE: echo-bot
REGION: us-central1
URL: https://echo-bot-1041791582068.us-central1.run.app
LAST DEPLOYED BY: Raimondo.Riccardi@eurecom.fr
LAST DEPLOYED AT: 2026-10-18T17:42:09.913127Z
<!--/answer-->

Now explain what that output tells you about how your Function is actually being run, why its
cold start resembles Cloud Run's so closely, and what — if anything — genuinely differs between
the two deployment paths. (~120 words)

<!--answer:convergence_explanation-->
The `gcloud run services list` output shows that Google Cloud Functions (Gen 2) executes directly on top of Cloud Run, managed behind the scenes by a system service account (`gcf-admin-robot`). This architectural convergence explains why their cold starts and warm latencies match so closely: both deploy and scale containerized workloads using the exact same underlying serverless infrastructure. 

The main difference lies in the deployment experience and control. Cloud Functions abstracts away container management, requiring only source code or a function package, whereas Cloud Run gives you explicit, direct control over custom container images, port configurations, and fine-grained deployment flags.
<!--/answer-->
