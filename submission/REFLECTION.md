# Day 23 Lab Reflection

> Fill in each section. Grader reads the "What I'd change" paragraph closest.

**Student:** Phạm Đình Trường
**Submission date:** 2026-05-12
**Lab repo URL:** https://github.com/phandinhTruong04/Day23-Track2-Observability-Lab

---

## 1. Hardware + setup output

Paste output of `python3 00-setup/verify-docker.py`:

```
Docker:        OK  (29.4.0)
Compose v2:    OK  (5.1.1)
RAM available: 7.62 GB (OK)
Ports free:    BOUND: [8000, 9090, 9093, 3000, 3100, 16686, 4317, 4318, 8888]
```

---

## 2. Track 02 — Dashboards & Alerts

### 6 essential panels (screenshot)

![AI Service Overview](file:///d:/Lab23/Day23-Track2-Observability-Lab/submission/screenshots/overview-dashboard.png)

### Burn-rate panel

![SLO Burn Rate](file:///d:/Lab23/Day23-Track2-Observability-Lab/submission/screenshots/slo-burn-rate.png)

### Alert fire + resolve

| When | What | Evidence |
|---|---|---|
| _T0_ | killed `day23-app`         | [alertmanager.png](file:///d:/Lab23/Day23-Track2-Observability-Lab/submission/screenshots/alertmanager.png) |
| _T0+90s_ | `ServiceDown` fired   | [slack-firing.png](file:///d:/Lab23/Day23-Track2-Observability-Lab/submission/screenshots/slack-firing.png) |
| _T1_ | restored app              | — |
| _T1+60s_ | alert resolved        | [slack-resolved.png](file:///d:/Lab23/Day23-Track2-Observability-Lab/submission/screenshots/slack-resolved.png) |

### One thing surprised me about Prometheus / Grafana

One thing that surprised me was the **power of the OTLP Collector**. I previously thought you had to point your app directly to Jaeger/Prometheus, but the Collector acts as a universal buffer. It allows us to send all telemetry (metrics, traces, logs) to a single endpoint and then route it to different backends (Loki, Jaeger, Prometheus) with advanced policies like **Tail-sampling** (keeping 100% of errors but only 1% of healthy traces) without ever touching the application code. This separation of concerns makes the entire observability stack much more scalable and flexible.

---

## 3. Track 03 — Tracing & Logs

### One trace screenshot from Jaeger

Drop `submission/screenshots/jaeger-trace.png` showing `embed-text → vector-search → generate-tokens` spans. 

*Self-check:* The flame graph clearly shows the `predict` parent span with three serial children: `embed-text` (5ms), `vector-search` (10ms), and `generate-tokens` (variable latency). Attributes like `gen_ai.usage.input_tokens` are visible in the `generate-tokens` span.

### Log line correlated to trace

Paste the log line and the trace_id it links to:

```json
{"event": "prediction served", "level": "info", "model": "llama3-mock", "input_tokens": 4, "output_tokens": 12, "quality": 0.793, "duration_seconds": 0.2858, "trace_id": "7520511446679d8362cd6e4f533c2689", "span_id": "cccf1bbb3b483751", "timestamp": "2026-05-11T11:43:46.158377Z"}
```
**Correlated Trace ID:** `7520511446679d8362cd6e4f533c2689`

### Tail-sampling math

The collector uses a composite policy:
1. `keep-errors`: 100%
2. `keep-slow` (>2s): 100%
3. `probabilistic-1pct`: 1%

For a service producing **100 traces/sec** with a 1% error rate and 1% slow rate:
- **Error traces:** 100 * 0.01 = 1.0/s
- **Slow traces (non-error):** 100 * 0.01 = 1.0/s
- **Healthy traces:** 100 * 0.98 = 98.0/s. Of these, 1% are kept: 98 * 0.01 = 0.98/s

**Total kept:** 1.0 + 1.0 + 0.98 = **2.98 traces/sec**
**Effective retention rate:** ~3%
**Storage savings:** ~97% compared to full ingestion.

---

## 4. Track 04 — Drift Detection

### PSI scores

Paste `04-drift-detection/reports/drift-summary.json`:

```json
{
  "prompt_length": {
    "psi": 3.461,
    "kl": 1.7982,
    "ks_stat": 0.702,
    "ks_pvalue": 0.0,
    "drift": "yes"
  },
  "embedding_norm": {
    "psi": 0.0187,
    "kl": 0.0324,
    "ks_stat": 0.052,
    "ks_pvalue": 0.133853,
    "drift": "no"
  },
  "response_length": {
    "psi": 0.0162,
    "kl": 0.0178,
    "ks_stat": 0.056,
    "ks_pvalue": 0.086899,
    "drift": "no"
  },
  "response_quality": {
    "psi": 8.8486,
    "kl": 13.5011,
    "ks_stat": 0.941,
    "ks_pvalue": 0.0,
    "drift": "yes"
  }
}
```

### Which test fits which feature?

For each of `prompt_length`, `embedding_norm`, `response_length`, `response_quality`, name the test (PSI / KL / KS / MMD) you'd choose in production and why.

1. **`prompt_length` -> PSI**: PSI is the industry standard for monitoring stability in feature distributions. It uses a binned approach that is robust to individual outliers while clearly highlighting overall shifts in user behavior (e.g., users starting to send much longer prompts).
2. **`embedding_norm` -> MMD**: Since embedding norms are sensitive indicators of semantic space shifts, **MMD (Maximum Mean Discrepancy)** is ideal as it captures changes in the distribution's moments using a kernel-based approach, which is more powerful than simple binning for high-dimensional representations.
3. **`response_length` -> KL Divergence**: KL Divergence is excellent for quantifying the relative entropy between distributions. It’s particularly useful for response lengths where we want to measure how much the "information profile" of our model's output has changed compared to the gold-standard baseline.
4. **`response_quality` -> KS Test**: The **Kolmogorov-Smirnov** test is a non-parametric test that is extremely sensitive to shifts in the CDF. For quality scores (which are bounded [0,1] and often skewed), KS can detect subtle shifts in the distribution's center or spread more reliably than binned metrics.

---

## 5. Track 05 — Cross-Day Integration

### Which prior-day metric was hardest to expose? Why?

The metric from **Day 20 (llama.cpp)** was the hardest to expose. Unlike modern web frameworks (FastAPI) or databases (Qdrant), llama.cpp is a low-level C++ engine that does not natively provide a Prometheus-formatted `/metrics` endpoint. Monitoring its internal performance (like tokens/sec) required a custom exporter or a "bridge" script to translate its API responses into a format Prometheus could scrape, highlighting the friction of instrumenting legacy or high-performance C++ systems.

---

## 6. The single change that mattered most

> **Grader reads this closest.**

The single most impactful change in this stack was the **implementation of the custom `structlog` processor for Trace Context Injection**. Initially, logs and traces were independent streams of data. By automatically injecting the `trace_id` and `span_id` from the OpenTelemetry context into every structured JSON log line, we transformed isolated logs into an integrated observability system. 

This change, combined with the Grafana **Derived Fields** configuration, enables "one-click" navigation from a slow inference log in Loki directly to the corresponding flame graph in Jaeger. This correlation is what makes the stack truly "useful" for production debugging rather than just a collection of charts. It directly implements the concept of **Unification via Correlation**, which is the most effective way to reduce Mean Time To Resolution (MTTR) when an AI service starts failing or slowing down.
