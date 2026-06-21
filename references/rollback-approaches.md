# Rollback Approaches

Use this reference when the user confirms the plan should include a rollback section. Walk through the relevant categories to identify the right approach.

## When to include rollback

Ask the user whether the plan needs a rollback section. If yes, use the questions below to identify which categories apply, then include the matching approaches in the plan.

## Rollback questioning flow

1. Does this change touch persistent state (database schema, stored data, file storage)?
2. Does this change affect APIs consumed by other services or clients?
3. Does this change alter deployment artifacts (containers, static assets, infrastructure)?
4. Does this change modify configuration or environment variables?
5. Does this change involve long-running workflows, pipelines, or state machines?
6. Does this change affect frontend assets served to end users?

For each "yes," pull in the matching category below.

---

## 1. Database rollback

### Schema migrations (DDL)

**Expand/contract pattern**
- Deploy in phases: add new → migrate data → update app → drop old
- When: production systems with zero-downtime requirements
- Trade-off: slower, requires discipline across multiple deploys
- Fails when: teams skip the contract phase or the expand step has subtle bugs

**Reverse migration scripts**
- Every `up` migration has a matching `down`
- When: early-stage products, staging environments, or controlled consumers
- Trade-off: `down` scripts are rarely tested; complex migrations often can't be cleanly reversed
- Fails when: `down` destroys data created after `up` ran

**Snapshot/restore**
- Take DB snapshot before migration, restore if needed
- When: small databases, maintenance windows acceptable
- Trade-off: restoring loses all data written after snapshot
- Fails when: database is large or data loss is unacceptable

**Shadow/dual-write**
- Write to old and new schema simultaneously, read from old, flip reads when validated
- When: high-risk schema changes on critical-path tables
- Trade-off: complex application logic, consistency risks
- Fails when: write patterns diverge subtly or dual-write code rots

### Data migrations

**Backfill with kill switch**
- Run backfill as background job with a flag to pause/abort
- When: populating new columns, reformatting existing data
- Trade-off: partial completion leaves data in mixed state
- Fails when: no idempotency — re-running creates duplicates

**Versioned data formats**
- Store a version field alongside data; app handles multiple versions
- When: document stores, event payloads, serialized blobs
- Trade-off: code complexity accumulates
- Fails when: old version handling is untested

---

## 2. API rollback

**URL-path versioning (/v1/, /v2/)**
- When: public APIs, breaking changes, long consumer migration timelines
- Trade-off: maintaining multiple versions is expensive
- Fails when: old versions never get sunset

**Additive-only changes (backward compatible)**
- Only add fields/endpoints, never remove or rename
- When: internal APIs with many consumers
- Trade-off: API surface bloats over time
- Fails when: someone renames a field assuming no one uses the old name

**Gateway-level rollback**
- API gateway routes traffic to old version when new version errors spike
- When: you have a gateway and version-stamped backends
- Trade-off: requires health-check integration; stateful APIs may diverge
- Fails when: the error is slow data corruption, not a clean failure

**Consumer-driven contract tests**
- Consumers define expectations; provider validates before deploy
- When: microservice ecosystems with decoupled deploy cadences
- Trade-off: setup cost, requires organizational buy-in
- Fails when: contracts don't cover edge cases

---

## 3. Deployment rollback

**Blue-green deployment**
- Two identical environments; swap traffic between them
- When: you can afford 2x infrastructure and need instant rollback
- Trade-off: cost; shared database state is still separate problem
- Fails when: new environment wrote data the old one doesn't understand

**Canary releases**
- Route small % of traffic to new version, watch metrics, promote or roll back
- When: high-traffic services needing production signal before full rollout
- Trade-off: requires good observability; canary may not exercise all code paths
- Fails when: bug only manifests at full load

**Rolling deployment**
- Replace instances one at a time; stop if health checks fail
- When: stateless services behind a load balancer
- Trade-off: old and new versions coexist during rollout
- Fails when: version incompatibility causes intermittent errors during the roll

**Feature flags**
- Deploy code dark, enable via flag, disable instantly if broken
- When: decoupling deploy from release
- Trade-off: flag debt accumulates; combinatorial testing is hard
- Fails when: flag-off path is untested or feature wrote data flag-off can't handle

**Immutable artifact rollback**
- Redeploy previous known-good container/AMI/artifact
- When: containerized or image-based deployments
- Trade-off: doesn't address database or state changes
- Fails when: artifact registry was pruned or infrastructure changed underneath

---

## 4. Frontend/UI rollback

**CDN cache invalidation + redeploy previous bundle**
- Push previous static assets, bust CDN cache
- When: SPA or static site deployments
- Trade-off: cache propagation delay
- Fails when: API version changed alongside frontend — rollback creates mismatch

**Client-side feature flags**
- Toggle UI features via remote config
- When: incremental UI changes, A/B tests
- Trade-off: flag evaluation adds latency; FOUC risk
- Fails when: flagged feature touched shared CSS/layout

**SSR fallback**
- Fall back to server-side rendering when client-side rendering breaks
- When: critical pages that must always render
- Trade-off: SSR infrastructure must be maintained in parallel
- Fails when: SSR path has its own bugs from lack of exercise

---

## 5. Configuration rollback

**Version-controlled config (GitOps)**
- Config in git; rollback = revert commit
- When: infrastructure config, feature flags, environment variables
- Trade-off: secret management needs separate tooling
- Fails when: config change triggered a side effect that revert doesn't undo

**Config service with audit trail**
- Central config service (Consul, etcd, Parameter Store) with change history
- When: runtime config that needs to change without redeploy
- Trade-off: config drift if not reconciled with code
- Fails when: propagation is eventually consistent and some nodes get old values

---

## 6. State machine / workflow rollback

**Compensating transactions (saga pattern)**
- Each step has an explicit undo action; on failure, run compensations in reverse
- When: distributed transactions across services
- Trade-off: compensations can themselves fail; eventual consistency window
- Fails when: compensation is not idempotent or external system doesn't support reversal

**Workflow versioning**
- In-flight workflows continue on old version; new workflows use new version
- When: long-running workflows where you can't migrate mid-flight
- Trade-off: must support N versions simultaneously
- Fails when: old version's dependencies are removed before all instances complete

**Checkpoint/replay**
- Persist state at each step; on failure, replay from last good checkpoint
- When: data pipelines, ETL jobs, batch processing
- Trade-off: checkpoint storage cost; replay must be idempotent
- Fails when: input data changed between original run and replay

---

## 7. Infrastructure rollback

**IaC revert (Terraform, Pulumi, CloudFormation)**
- Revert the IaC commit and apply
- When: any IaC-managed infrastructure
- Trade-off: some resources can't be recreated; destroy-then-create ordering can cause downtime
- Fails when: state file is out of sync with reality

**Immutable infrastructure**
- Bake new image, replace instances entirely
- When: eliminating config drift
- Trade-off: slower deploy cycle
- Fails when: state lives on the instance and isn't externalized

**DNS failover**
- Shift DNS to standby environment or region
- When: disaster recovery, region-level failures
- Trade-off: DNS TTL means propagation delay
- Fails when: standby environment data is stale

**Container orchestration rollback**
- Revert to previous deployment revision
- When: Kubernetes or similar orchestrator deployments
- Trade-off: only rolls back pod spec, not config or secrets
- Fails when: issue is in config, not the container image

---

## Cross-cutting principles

- **Rollback is not the inverse of deploy.** Deploys are additive; rollbacks must handle data/state written by the new version.
- **Test the rollback path.** Most rollback failures happen because rollback was never exercised.
- **Decouple deploy from release.** Feature flags and traffic shifting give release rollback without deploy rollback.
- **Idempotency is the foundation.** Every rollback mechanism depends on operations being safely re-runnable.
- **Observability determines rollback speed.** You can only roll back as fast as you can detect the problem.
- **Partial rollback is the hardest case.** Rolling back one layer while others stay forward is where most incidents escalate.
