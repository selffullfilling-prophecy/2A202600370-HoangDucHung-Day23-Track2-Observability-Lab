# Day 23 Lab Reflection

**Student:** Hoang Duc Hung - 2A202600370
**Submission date:** 2026-05-11  
**Lab repo URL:** not provided in the local workspace

---

## 1. Hardware + setup output

Output of `python 00-setup/verify-docker.py` after the stack was running:

```text
Docker:        OK  (29.4.0)
Compose v2:    OK  (5.1.2)
RAM available: 7.43 GB (OK)
Ports free:    BOUND: [8000, 9090, 9093, 3000, 3100, 16686, 4317, 4318, 8888]
Report written: C:\Users\Admin\Documents\2A202600370-HoangDucHung-Day23-Track2-Observability-Lab\00-setup\setup-report.json
```

The ports are expected to be bound because the observability stack is already up.

---

## 2. Track 02 - Dashboards & Alerts

### 6 essential panels

Screenshot: `submission/screenshots/dashboard-overview.png`.

### Burn-rate panel

Screenshot: `submission/screenshots/slo-burn-rate.png`.

### Cost and tokens

Screenshot: `submission/screenshots/cost-and-tokens.png`.

### Alert fire + resolve

| When | What | Evidence |
|---|---|---|
| 2026-05-11T14:19Z | stopped `day23-app` | `submission/screenshots/alertmanager-firing.png` |
| 2026-05-11T14:19Z | `ServiceDown` fired in Alertmanager | `submission/screenshots/alertmanager-firing.png` |
| 2026-05-11T14:22Z | restored app | `submission/screenshots/alertmanager-resolved.png` |
| 2026-05-11T14:23Z | Alertmanager had no active alerts | `submission/screenshots/alertmanager-resolved.png` |

Slack fire/resolve UI screenshots require authenticated access to the Slack workspace, so I documented that gap in `note.md`.

### One thing surprised me about Prometheus / Grafana

The most brittle part was not the PromQL itself, but stable datasource wiring. Grafana generated random datasource UIDs until I pinned Prometheus to `uid: prometheus`, which made dashboard-as-code reproducible across a fresh volume.

---

## 3. Track 03 - Tracing & Logs

### One trace screenshot from Jaeger

Screenshot: `submission/screenshots/jaeger-trace.png`.  
GenAI span attributes screenshot: `submission/screenshots/jaeger-attrs.png`.

The trace `71059717261f524c3f8946f54924d94a` shows `POST /predict` -> `predict` -> `embed-text`, `vector-search`, and `generate-tokens`.

### Log line correlated to trace

```json
{"model":"llama3-mock","input_tokens":5,"output_tokens":32,"quality":0.777,"duration_seconds":0.3345,"trace_id":"1a29f79aa286852720c93e4b191462db","event":"prediction served","level":"info","timestamp":"2026-05-11T14:09:06.807863Z"}
```

### Tail-sampling math

During the main load run I sent 241 successful requests and 1 forced error request. The collector policy keeps all ERROR traces, all traces slower than 2000 ms, and 1% of healthy traces. For healthy traffic, the retained fraction is:

```text
241 healthy traces * 0.01 = about 2.4 retained healthy traces
```

The forced error trace is retained deterministically because the app sets the `predict` span status to ERROR before returning HTTP 503.

---

## 4. Track 04 - Drift Detection

### PSI scores

`04-drift-detection/reports/drift-summary.json`:

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

Evidently report screenshot: `submission/screenshots/drift-report.png`.

### Which test fits which feature?

`prompt_length`: PSI for dashboarding and KS for alert confirmation. It is numeric, easy to bucket, and the synthetic shift is a location/shape change.

`embedding_norm`: KS is a good first check because it catches distribution changes without assuming a specific parametric form. For full embeddings in production, MMD is better because it compares high-dimensional distributions instead of a single norm.

`response_length`: PSI is useful operationally because length buckets are easy to explain to product and cost stakeholders. KS is useful as a statistical backup when the distribution shape changes without crossing a coarse PSI bucket.

`response_quality`: KS is the strongest fit for numeric eval scores in [0,1]. KL is useful when comparing full score histograms, but it is more sensitive to empty bins and needs smoothing.

---

## 5. Track 05 - Cross-Day Integration

Screenshot: `submission/screenshots/cross-day-dashboard.png`.

### Which prior-day metric was hardest to expose? Why?

The hardest prior-day metric would be Day 20 llama.cpp tokens/sec because serving stacks often expose different metric names or no Prometheus endpoint by default. The dashboard can fail soft with "No Data", but a production integration would need either a stable exporter or a small adapter that normalizes llama.cpp output into `day20_llamacpp_tokens_per_second`.

---

## 6. The single change that mattered most

The single change that mattered most was pinning observability contracts to stable names: metric names, bounded labels, datasource UIDs, dashboard UIDs, and trace span names. Before that, the stack could run but still be hard to grade or operate because a dashboard might point at a random datasource UID or a trace might split the parent `predict` span from its child spans. After the fix, the same Compose stack can be recreated and still produce the same dashboard and trace shape.

That maps directly to the deck's RED/USE and dashboard-as-code ideas: observability is only useful when the signal is stable enough to compare over time. A request counter, latency histogram, token counter, quality gauge, and trace attributes are small pieces, but together they form a contract between the app and the SRE workflow.
