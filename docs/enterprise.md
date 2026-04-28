# Community vs Enterprise


The following are **Enterprise-only** and physically absent from the CE
binary (their source isn't compiled in):

- Multi-tenancy
- Cluster mode (Raft consensus, replication, failover)
- Change Data Capture (CDC) / binlog ingest / Kafka source-sink
- SSO/SAML/OIDC, LDAP/AD
- Column-level and row-level access policies
- Encryption at rest (KMS-backed)
- Audit log retention beyond 30 days, SIEM export
- Scheduled backups, point-in-time recovery, cross-region backup
- Async ML training workers, model deployment lifecycle
- Background transcription worker for long-form audio/video
- Threat detection
- Federated query (`CREATE CONNECTION` to external databases)
- Streaming SQL (continuous queries with window operators)
- Graph CDC

If a customer is asking for any of the above, that's an Enterprise
conversation.

