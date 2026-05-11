# Day 23 Lab Reflection

**Student:** phucngo  
**Submission date:** 2026-05-11  
**Lab repo URL:** local submission

---

## 1. Hardware + setup output

Output of `python 00-setup/verify-docker.py` while the stack was running:

```text
Docker:        OK  (29.0.1)
Compose v2:    OK  (2.40.3-desktop.1)
RAM available: 3.74 GB (NEED >= 4.0 GB)
Ports free:    BOUND: [8000, 9090, 9093, 3000, 3100, 16686, 4317, 4318, 8888]
Report written: C:\Users\admin\Desktop\New folder (12)\lap23\Day23-Track2-Observability-Lab\00-setup\setup-report.json
```

The RAM warning is expected on this machine. The port warning appears because the Docker stack is already running and has bound the lab ports.

---

## 2. Track 02 - Dashboards & Alerts

Dashboard evidence:

- `submission/screenshots/02-grafana-overview.png`
- `submission/screenshots/03-grafana-slo.png`
- `submission/screenshots/04-grafana-cost.png`

Alert evidence:

| When | What | Evidence |
|---|---|---|
| T0 | stopped `day23-app` | `submission/screenshots/05-alertmanager.png` |
| T0+65s | `ServiceDown` fired | `submission/screenshots/06-slack-fire.png` |
| T1 | restarted `day23-app` | local Docker run |
| T1+30s | alert resolved | `submission/screenshots/07-slack-resolve.png` |

One thing that surprised me about Prometheus/Grafana is how much the alert `for:` duration affects the demo. The app was down immediately, but the useful alert only fired after Prometheus confirmed the condition for the configured window. That delay is the difference between alerting on real sustained failures and alerting on a short scrape glitch.

---

## 3. Track 03 - Tracing & Logs

Jaeger evidence:

- `submission/screenshots/08-jaeger-trace.png`
- `submission/screenshots/09-jaeger-attrs.png`

Correlated JSON log line:

```json
{"model": "llama3-mock", "input_tokens": 4, "output_tokens": 27, "quality": 0.827, "duration_seconds": 2.0457, "trace_id": "1dcb0f06e87898aa43ea98ca96affd2f", "event": "prediction served", "level": "info", "timestamp": "2026-05-11T12:40:39.807046Z"}
```

Tail-sampling math: the collector keeps 100% of error traces, 100% of traces slower than 2000 ms, and 1% of normal healthy traces. In the screenshot trace, the request lasted about 2.04 s, so it was retained by the latency policy. If the service produced 100 healthy traces/sec and no errors/slow requests, about 1 trace/sec would be kept by the 1% probabilistic policy.

---

## 4. Track 04 - Drift Detection

PSI/KL/KS output from `04-drift-detection/reports/drift-summary.json`:

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

For `prompt_length`, PSI is a good production test because it is easy to explain as distribution shift across buckets. For `embedding_norm`, KS is useful because it catches continuous distribution movement without requiring fixed business buckets. For `response_length`, PSI or KS both work; I would use KS for statistical sensitivity and PSI for dashboard readability. For `response_quality`, PSI is best for operational alerting because the quality drop is the business-facing symptom, while KL can be kept as a diagnostic metric.

---

## 5. Track 05 - Cross-Day Integration

Cross-day evidence:

- `submission/screenshots/11-cross-day-dashboard.png`
- Prometheus scraped `day19_qdrant_collections=3` from `day19-stub`
- Prometheus scraped `day20_llamacpp_tokens_per_second` from `day20-stub`

The hardest prior-day metric to expose would be llama.cpp tokens/sec because the model server does not always expose Prometheus metrics natively. It often needs a sidecar, wrapper, or patched server endpoint, while Qdrant and infrastructure exporters are more standard.

---

## 6. The single change that mattered most

The most important change was fixing the trace structure so `predict` is the current parent span and `embed-text`, `vector-search`, and `generate-tokens` become real child spans in the same trace. Before that, Jaeger showed separate spans, which technically proved spans existed but did not explain one request end to end.

That change made the observability stack useful instead of just present. It connects the high-level request metric to the actual slow component, and it matches the deck idea that traces answer "where did the time go?" while metrics answer "how often is this happening?"
