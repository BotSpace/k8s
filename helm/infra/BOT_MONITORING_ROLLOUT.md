# Engine pool monitoring and conversation rollout

This change requires compatible **backend API + backend worker, engine and admin images**. Updating only this manifest does not install the new binaries. Image tags remain unchanged until CI publishes the corresponding images; promote their real immutable tags through the existing GitOps workflow. Do not apply a placeholder tag.

## Preflight

- Back up PostgreSQL, CMS MongoDB and pool Redis using the normal operational process.
- `backend-golang-app` must provide `APP_MONGO_URL` and `APP_ENGINE_MONGO_DB_NAME`. API, backend worker, pool supervisor and children must use this same CMS destination. The pool no longer chooses the legacy `APP_ENGINE_MONGO_ADDR` destination. If these previously differed, plan the CMS data migration first; this rollout does not move data.
- API/worker/pool must use the same pool Redis instance/database. Existing pool deployment uses Redis database 0. Do not point the backend worker at an unrelated task Redis database.
- Redis conversation outbox streams must be persistent (AOF/durable volumes, appropriate replication) and **not evicted**. Verify the external Redis service policy; this repository does not manage its persistence configuration. Plan capacity monitoring because pending streams are not trimmed.
- Check MongoDB collection `bot_conversations` indexes. Engine startup rejects an obsolete project/update-only unique index or another conflicting unique constraint rather than silently losing events. Follow `engine-golang/docs/BOT_MONITORING.md` index-upgrade procedure after backup; do not blindly drop indexes or history.
- Backend `APP_ENGINE_NAMESPACE` must match `botmother`; its Prometheus URL must point at the Prometheus instance selecting `release: monitoring` ServiceMonitors.

## Rollout order

1. Deploy the new backend **API and worker** first. Existing worker startup/periodic reconciliation adds the authoritative canonical UUID to stored runtime specs, including numeric legacy tenants. Verify `pool runtime identity reconciliation checked` has `failed=0` and the expected runtime count. Never print full Redis runtime values: they contain credentials. Unmapped/conflicting projects need authoritative SQL correction or a normal backend redeploy.
2. Publish/promote the new engine image, then sync `engine-pool.yaml`. Supervisor and child startup now require working conversation persistence. Use the usual rolling window: spec migration can restart children.
3. Sync `monitoring.yaml`. The `engine-pool` ServiceMonitor selects the standalone Service by `app: botmother-engine-pool`, scrapes **each worker endpoint** at `/metrics` on the named `http` port every 15 seconds, and attaches its Kubernetes namespace. No extra per-child scrape or per-project Service is needed.
4. Deploy the new admin image.

## Acceptance checks

- All expected `engine-pool` Prometheus targets are UP; do not mistake one healthy load-balanced Service response for all healthy workers.
- For a running bot, query `engine_pool_runtime_memory_rss_bytes{namespace="botmother"}` and `rate(engine_pool_runtime_cpu_seconds_total{namespace="botmother"}[5m])`. Labels must carry the canonical SQL project UUID, not a numeric CMS tenant or webhook UUID. CPU needs at least two scrapes. Sleeping bots may legitimately have no live process samples.
- In admin, inspect CPU/RAM and history. Missing samples must show unavailable, not zero. Shared worker network traffic is not per-bot traffic and is deliberately not fabricated.
- Send a test message and callback to a bot and trigger its reply. In MongoDB, verify inbound/outbound records in `bot_conversations`, UTC BSON `created_at`, canonical `project_id`, and redacted payloads. Replay the same inbound update in a controlled test: it must not create another inbound record.
- Exercise a custom-code send, inline edit and copy action. Outbound payloads contain allowlisted request fields plus Telegram results; copied content absent from both is represented by source references, not invented text.
- Check bot total users, rolling 24h/7d active users and 24h events. Test activity/user sorting in both directions with owner/date/template/release filters across multiple pages. Overall user totals are bot-user memberships, not globally unique people.
- Stop a test bot and verify its process metrics disappear and no orphan assignment restarts it. Pending outbound events must still drain via a remaining worker.

## Data and safety boundaries

History starts when recording is deployed; Telegram does not backfill earlier conversations. Media references/metadata are retained, not downloaded media binaries. Simulator traffic is not production history. There is no implicit TTL: set an explicit retention/deletion policy for this sensitive user data, Mongo backups and pending Redis events. Access should be restricted and collection disclosed to bot users.

Mongo failure rejects inbound webhook delivery for upstream retry. Successful outbound responses wait for Redis enqueue and are retried into Mongo. Telegram acceptance and Redis enqueue cannot be atomic, so a crash in that gap can still lose an outbound event; this is not exactly-once delivery.

These manifests have not been applied to a live cluster by the coding agent. Validate rollout and authenticated UI behavior in staging before production.
