# Limits & quotas


CE has hard-coded scale caps. They live as `pub const` values in
`crates/aidb-licensing/src/community_limits.rs` — there is no config
key or env var that lifts them, by design.

| Dimension                           | CE cap          |
|-------------------------------------|-----------------|
| Tenants per deployment              | 1               |
| Dashboards per tenant               | 5               |
| AutoML models per tenant            | 5               |
| AutoML training rows per run        | 1,000,000       |
| Continuous aggregates per tenant    | 5               |
| Streaming-insert throughput         | 1,000 rows/sec  |
| Audit log retention                 | 30 days         |
| External LLM calls / user / day     | 300             |
| NL2SQL calls / user / day           | 50              |
| AI chat messages / user / day       | 100             |
| Vectors per single collection       | 50,000,000      |
| Columnar storage per tenant         | 100 GB          |
| Streaming MATCH paths per session   | 10,000          |

To raise any of these, you need an Enterprise license. Drop the signed
license file at `/etc/synapcores/license.json` and run an Enterprise
build of the binary; the universal binary will read its caps from the
license payload.

To inspect the active limits at runtime:

```bash
curl -H "Authorization: Bearer $TOKEN" http://localhost:8080/v1/license
```

