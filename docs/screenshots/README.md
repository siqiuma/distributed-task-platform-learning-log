# Observability Screenshots

These screenshots demonstrate the observability pipeline implemented in this project:

- Application metrics → Prometheus
- Visualization via Grafana
- Alerting via Alertmanager → Slack

The system tracks:
- Task throughput
- Success vs failure rate
- Latency distributions (p50/p95/p99)
- Automated alerting for failure, latency, and stalled workers
