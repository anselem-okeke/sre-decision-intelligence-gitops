# Phase 10 — Incident Timeline

| Time | Event | Evidence |
|---|---|---|
| T0 | Frontend healthy | `probe_success = 1` |
| T1 | Frontend pods confirmed running | `frontend-67dd44c5c9-zsjc9 1/1 Running` |
| T2 | Frontend Service had valid endpoint | `10.244.8.229:8080` |
| T3 | Incident injected | Service selector patched with `slo-test=broken` |
| T4 | Frontend Service lost endpoints | `frontend <none>` |
| T5 | Probe detected failure | `probe_success = 0` |
| T6 | Availability SLO dropped | `avg_over_time(...) = 0.7` |
| T7 | Alert entered pending state | `BankOfAnthosFrontendAvailabilitySLOBreach alertstate="pending"` |
| T8 | Service selector restored | removed `/spec/selector/slo-test` |
| T9 | Frontend endpoint restored | `10.244.8.229:8080` |
| T10 | Probe recovered | `probe_success = 1` |
