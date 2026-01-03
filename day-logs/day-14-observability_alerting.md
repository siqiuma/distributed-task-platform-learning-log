# Day 14 — Observability & Alerting Completion

**Focus:** Production-grade observability, alerting, and operational maturity

---

## ✅ What I Accomplished Today

### 1. Metrics & Dashboards (Completed)
- Verified all custom task metrics are correctly exposed via `/actuator/prometheus`
- Confirmed Prometheus is scraping the application successfully
- Built a Grafana dashboard with:
  - Task processed rate (5m)
  - Success vs failure rate
  - Failure ratio
  - Latency p50 / p95 / p99
  - Latency histogram bucket distribution
- Investigated and understood why graphs move even without external requests (scheduler + background workers)
- Validated that all panels update correctly in real time

---

### 2. Alerting Pipeline (Completed End-to-End)
- Defined alert rules for:
  - Worker not processing tasks
  - High failure ratio
  - markFailed errors
  - High p95 latency
- Wired Prometheus → Alertmanager → Slack
- Debugged multiple real-world issues:
  - Alertmanager config path & volume mounting
  - Slack webhook URL formatting
  - Environment variable interpolation limitations
- Successfully accessed Alertmanager UI at `http://localhost:9093`
- Verified alerts appear in Alertmanager UI
- Confirmed **real alerts were delivered to Slack**
  - End-to-end pipeline is fully functional

---

## 🧠 Key Learnings

- Metrics can change even without HTTP traffic due to background workers and schedulers
- Histogram metrics are best understood via:
  - `histogram_quantile(...)` for p95/p99
  - Bucket visualizations for distribution shape
- Alerting without delivery (Slack/Pager) is incomplete
- Small config details (paths, env vars, flags) are common real-world failure points
- Observability work strongly signals senior-level backend / infra experience

---

## 📦 Deliverables Now Complete

- ✅ Prometheus metrics
- ✅ Grafana dashboards
- ✅ Alert rules
- ✅ Alertmanager integration
- ✅ Slack notifications
- ✅ Verification via UI + real messages

