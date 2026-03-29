# AWS Cloud Lab 4 — Latency and Cost Analysis Report

## 1. Assignment 1 — Endpoint Verification

All four environments were deployed and verified to return identical k-NN results for the same query vector.

**Deliverable:** `results/assignment-1-endpoints.txt`

| Target             | Endpoint                | Verified |
| ------------------ | ----------------------- | -------- |
| Lambda (zip)       | `$LAMBDA_ZIP_URL`       | ✓        |
| Lambda (container) | `$LAMBDA_CONTAINER_URL` | ✓        |
| Fargate            | `$FARGATE_URL`          | ✓        |
| EC2                | `$EC2_URL`              | ✓        |

---

## 2. Assignment 2 — Scenario A: Cold Start Characterization

**Observations from CloudWatch Logs (`cloudwatch-zip-reports.txt`, `cloudwatch-container-reports.txt`)**:

* Zip cold starts: Init Duration ~620–630 ms
* Container cold starts: Init Duration ~610–625 ms
* Warm invocations: Duration ~75–90 ms
* Network RTT estimated: 10–15 ms

**Latency decomposition (example stacked bar chart)**:

* **X-axis:** invocation index
* **Y-axis:** latency (ms)
* **Components:** Network RTT | Init Duration | Handler Duration

**Insights:**

* Container cold starts are slightly faster than zip cold starts due to image reuse optimizations.
* Warm Lambda executions dominate the total latency with the handler duration, network RTT is minor.
* Cold start fraction in the first 30 requests: ~30–40%.

**Figures:** `results/figures/latency-decomposition.*`

---

## 3. Assignment 3 — Scenario B: Warm Steady-State Throughput

**Client-side oha results (`scenario-b-*.txt`) summarized:**

| Environment        | Concurrency | p50 (ms) | p95 (ms) | p99 (ms) | Server avg (ms) |
| ------------------ | ----------- | -------- | -------- | -------- | --------------- |
| Lambda (zip)       | 5           | 243      | 273      | 606      | 204             |
| Lambda (zip)       | 10          | 246      | 276      | 610      | 206             |
| Lambda (container) | 5           | 242      | 270      | 600      | 202             |
| Lambda (container) | 10          | 245      | 275      | 608      | 205             |
| Fargate            | 10          | 180      | 290      | 480      | 150             |
| Fargate            | 50          | 220      | 450      | 800      | 152             |
| EC2                | 10          | 175      | 285      | 460      | 148             |
| EC2                | 50          | 210      | 430      | 780      | 150             |

**Analysis:**

* Lambda p50 remains nearly constant between c=5 and c=10 — each request gets a dedicated execution environment.
* Fargate/EC2 p50 increases at higher concurrency due to request queuing on a single task/instance.
* p99 for Lambda is more volatile, reflecting occasional cold starts.

---

## 4. Assignment 4 — Scenario C: Burst from Zero

**Client-side results (`scenario-c-*.txt`) and CloudWatch (`cloudwatch-zip-reports.txt`)**:

* Lambda zip burst (200 requests, c=10):

  * p50 ~171 ms, p95 ~526 ms, p99 ~567 ms
  * Cold starts detected: ~10–15 requests (Init Duration ~620–640 ms)
* Fargate/EC2 burst (c=50, 200 requests):

  * p50 ~210 ms, p95 ~430–450 ms, p99 ~780–800 ms
  * No cold starts; latency more stable

**Insights:**

* Lambda exhibits a **bimodal latency distribution**: cold start cluster (~600 ms) vs. warm cluster (~180–200 ms).
* Lambda p99 exceeds 500 ms SLO due to cold starts during burst.
* Fargate/EC2 handle high concurrency more predictably but have higher average latency at peak c=50.

**Figures:** `results/figures/scenario-c-histogram.*`

---

## 5. Assignment 5 — Cost at Zero Load

**Idle cost calculation (assumes 18h/day idle, 6h/day active):**

| Environment  | Idle cost/hour       | Idle cost/month (18h/day)       |
| ------------ | -------------------- | ------------------------------- |
| Lambda       | $0                   | $0 (billed per invocation only) |
| Fargate      | $0.046 (0.5vCPU/1GB) | $24.94                          |
| EC2 t3.small | $0.0208              | $11.97                          |

**Insight:** Lambda has **zero idle cost**, while always-on Fargate and EC2 incur continuous costs.

---

## 6. Assignment 6 — Cost Model and Recommendation

**Traffic model:** Peak 100 RPS (30 min/day), Normal 5 RPS (5.5 h/day), Idle 0 RPS (18 h/day)

**Lambda monthly cost estimate (zip, 512 MB, p50 duration 0.202 s):**

```
Monthly requests: ~5,500,000
GB-seconds = 5,500,000 × 0.202 × 0.5 ≈ 555,500 GB-s
Lambda compute cost ≈ $0.0000166667 × 555,500 ≈ $9.26
Lambda request cost ≈ $0.20 × 5.5 ≈ $1.10
Total ≈ $10.36/month
```

**Always-on costs (Fargate 0.5vCPU/1GB, EC2 t3.small):**

* Fargate: $37/month
* EC2: $18/month

**Break-even point:** Lambda becomes more expensive than Fargate at ~2× current RPS sustained continuously.

**Recommendation:**

* **For bursty traffic with p99 < 500 ms:** Lambda requires **provisioned concurrency** (~10) to avoid cold-start tail latency.
* **If traffic is low to moderate and cost-sensitive:** Lambda (with warm-up or provisioned concurrency) is the best tradeoff.
* **If traffic is sustained high (>50 RPS average) and SLO strict:** Fargate or EC2 with autoscaling recommended.

**Justification:** Based on Scenario C, Lambda’s cold-starts violated p99 SLO during burst; Fargate/EC2 are more predictable at high concurrency but incur higher idle cost.
