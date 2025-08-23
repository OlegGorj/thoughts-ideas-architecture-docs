Here’s a crisp, Day-2–focused gameplan for **HA, DR, failover, and recovery** of Azure data tiers (PostgreSQL Flexible Server, SQL Managed Instance) that were provisioned via Terraform—plus how **Temporal** + **lifecycle separation** lets you do this safely without “collateral applies.”

---

# 0) Assumptions (explicit)

* **State backend:** Terraform remote state in Azure Storage (per tier: AKV, DB, App/VM). Storage uses GRS/RA-GRS for DR.
* **Tiers are decoupled:** separate Terraform roots & state for **AKV**, **DB**, **VMs**. VMs read DB connection via **AKV secret**; they don’t hardcode endpoints.
* **Temporal** runs the operational runbooks; Terraform is used for **desired config** (not for “press the red failover button” at runtime).
* **RTO/RPO** vary by tier; DB is the primary driver.
* **Triggers:** ad-hoc (CLI/UI/Alert) plus scheduled drills.

---

# 1) Azure capabilities you can (and should) leverage

## PostgreSQL Flexible Server (PG Flex)

* **In-region HA:** zone-redundant or same-zone HA (synchronous failover within region). Not available on Burstable; use GP/MO SKUs. ([Microsoft Learn][1])
* **Read replicas:** up to 5; **geo-replication** supported (async) for cross-region DR; promote for DR. ([Microsoft Learn][2])
* **Backups & PITR:** automatic backups (ZRS storage), PITR up to 35 days. ([Microsoft Learn][3])

## Azure SQL Managed Instance (SQL MI)

* **Built-in HA:** local + **zone-redundant** availability architecture. ([Microsoft Learn][4])
* **Cross-region DR:** **Failover groups (FOG)** between paired regions; group all user DBs, get listener endpoints, manual or automatic failover. ([Microsoft Learn][5])
* **Backups, PITR & LTR** (long-term retention) managed by the platform (not cited here; standard MI feature).

## Azure Key Vault (AKV)

* **High availability** as a service; **backup/restore** available for DR (region recovery strategy). ([Microsoft Learn][6])

> Terraform resources exist to **configure** these features (e.g., MI failover groups) but **operational actions** (promote replica, force failover, restore from backup) are runtime tasks via Azure APIs/CLI—not idempotent “config.” Use Terraform for desired topology; use Temporal for “Day-2 buttons.” (Ref: azurerm `mssql_managed_instance_failover_group`, `postgresql_flexible_server` docs). ([Terraform Registry][7])

---

# 2) Separation of lifecycles: why it matters on Day-2

* **DB failover** must **not** trigger a VM apply. Because VMs depend only on an **AKV secret** (not a hardcoded FQDN), swapping DB primaries = **just rotate the secret**; VM tier untouched.
* **AKV rotation** must **not** touch DB/VM resources.
* **VM rebuilds** must never alter DB/AKV.

This is exactly what separate Terraform states per tier give you; Temporal coordinates the **order** and **runtime side effects** (like secret updates, connection drains), not Terraform.

---

# 3) Patterns for HA/DR with Terraform (config) + Temporal (operations)

## Pattern A — “Configure for resilience, then let the platform auto-heal”

* Terraform **enables HA** knobs only:

  * PG Flex: enable in-region HA (zone-redundant if available), set PITR retention; optionally create a cross-region **read-replica** (cold/warm DR). ([Microsoft Learn][1])
  * SQL MI: enable **zone redundancy**; provision a **failover group** to secondary region. ([Microsoft Learn][4])
* Temporal is used for **readiness probes**, **failover toggles** (e.g., planned MI FOG switchover), and **post-failover hygiene** (rotate AKV secret, warm caches, cutover DNS).

**When to choose:** strict IaC hygiene, minimal Day-2 logic; rely on Azure HA/FOG.

## Pattern B — “Automated switchover/failover runbooks” (recommended)

* Terraform defines topology (primary, replicas/FOG, AKV secrets).
* Temporal owns **runbooks** for:

  * **Planned switchover** (zero/minimal data loss): drain writes, set read-only, switchover/promote, update AKV, app warm-up, un-drain.
  * **Unplanned failover** (DR): promote DR instance / FOG failover, rotate AKV, confirm app health, open circuit again.
  * **Failback**: reverse replication, resync, controlled cutback.
  * **Backup/PITR restore workflows**: create sidecar restored instance, run checks, then switch AKV to the restored endpoint.
* Temporal handles **retries, timeouts, compensations**, long-running steps—what it was built for (durable execution). ([Temporal][8], [Medium][9])

**When to choose:** you need push-button DR drills, documented, repeatable, observable.

## Pattern C — “Low-cost DR via backups”

* No cross-region replicas. Use **PITR** or **geo-restore** only.
* Temporal runbook for **region loss**: restore in DR region → rotate AKV → cutover apps → (optional) recreate former primary later.

**When to choose:** lower RTO budget, cost-optimized.

---

# 4) Concrete Temporal runbooks (tier-aware, no cross-tier applies)

Below are skeletal flows you can implement as Temporal workflows/activities. Steps with **Terraform** run as activities that **apply config**. Steps with **Azure API/CLI** are operational and **do not** touch Terraform state.

## 4.1 SQL Managed Instance — Planned Switchover (FOG)

1. **Prechecks**: health of both instances, FOG state, backlog < threshold.
2. **App drain**: signal app tier (via AKV flag or feature toggle) → switch to read-only mode, quiesce writes.
3. **FOG planned failover**: call Azure FOG **planned failover** API. ([Microsoft Learn][5])
4. **AKV update**: set **single connection secret** (e.g., `DB_CONN_URI`) to new primary listener.
5. **Warm-up & health**: run synthetic queries on new primary, warm plan cache.
6. **Resume writes**: flip app back to read-write.
7. **Observability**: emit metrics, annotate run in history.

> Terraform is used **earlier** to create/configure the FOG, not to do the switchover itself. ([Terraform Registry][7])

## 4.2 SQL Managed Instance — Unplanned Failover (DR)

1. **Detect**: heartbeat/alarm from MI region.
2. **Force failover**: FOG **forced failover** to secondary (accept potential data loss). ([Microsoft Learn][5])
3. **AKV secret rotation** to DR endpoint.
4. **App validation**; if not healthy, auto-rollback is **not** possible—engage incident bridge.
5. **Post-mortem hooks** and **failback planning** workflow scheduled.

## 4.3 PostgreSQL Flexible Server — Planned Switchover to Geo-Replica

1. **Prechecks**: replication lag, locks, extensions parity.
2. **App drain** to read-only.
3. **Promote replica** in DR region (planned, after ensuring sync lag ≈ 0). ([Microsoft Learn][10])
4. **AKV secret rotation** to promoted endpoint.
5. **DNS / App pool warm-up**; restore read-write.
6. **Re-establish replication** from new primary to former region (optional).
7. **Audit trail**.

> In-region HA (zone-redundant) is **automatic**; no runbook needed there except health checks. ([Microsoft Learn][1])

## 4.4 PostgreSQL Flexible Server — Region Loss (Backup/PITR)

1. **Declare DR**; pick last good recovery point (PITR). ([Microsoft Learn][3])
2. **Restore** to DR region.
3. **AKV update**, app warm-up, resume.
4. **Recreate replica** back to former primary region when recovered.

---

# 5) Where Terraform ends and Temporal begins (clear contract)

| Concern                                        | Terraform (config/state) | Temporal (Day-2 operations)  |
| ---------------------------------------------- | ------------------------ | ---------------------------- |
| Create MI/PG, SKUs, storage, VNet              | ✅                        | –                            |
| Enable **zone redundancy**                     | ✅ (MI, PG Flex)          | –                            |
| Configure **FOG** (MI) / **Geo-replicas** (PG) | ✅                        | –                            |
| Rotate **AKV** secret schema/permissions       | ✅ (policy)               | ✅ (write values on cutover)  |
| **Planned switchover / forced failover**       | –                        | ✅ (API/CLI)                  |
| **Restore/PITR**, self-healing                 | –                        | ✅                            |
| **Drift detection**                            | Plan only (no apply)     | ✅ notify, gate, or reconcile |
| **Failback** choreography                      | –                        | ✅                            |

> Practical note: after certain DR actions (e.g., restore to a *new* resource ID), **refresh/import** the Terraform state during the *next maintenance window* so IaC matches reality. Temporal can queue a “state reconciliation” task (read IDs, run `terraform import` or refactor module).

---

# 6) Drift, state & idempotency (post-failover hygiene)

* **Drift detection**: run scheduled `terraform plan` (no-apply) per tier; if changes are detected (exit code 2), **notify** on-call and link the Temporal run. Do **not** auto-apply in DR; human approval first.
* **After failover/restore**:

  * If endpoints changed, only **AKV secret value** changes; VM Terraform state remains unchanged (decoupled).
  * If **new resource IDs** (e.g., PG restored to new server ID), record **reconciliation debt**: next maintenance window runs a controlled `terraform import` or module refactor so desired state matches.
* **Idempotency**: Temporal activities use **idempotency keys** (e.g., switchover run ID) to avoid double-promotion if a retry happens.

---

# 7) Operational SLOs & routing

* **RTO/RPO** menu:

  * **Fastest:** MI **FOG** or PG geo-replica promotion (minutes RTO, RPO=replication lag). ([Microsoft Learn][5])
  * **Mid-range:** in-region HA for node loss (seconds/minutes, RPO≈0). ([Microsoft Learn][1])
  * **Cheapest:** backup/PITR (RTO=restore time, RPO=backup granularity). ([Microsoft Learn][3])
* **State backend DR:** use **RA-GRS** for Terraform state storage; document failover/secondary read & re-point plan. ([Microsoft Learn][11])
* **Temporal HA:** run Temporal (or Temporal Cloud) across AZs; use retries & heartbeats for long activities (DB restore can be hours). Temporal’s durability model is a good fit for long DB ops. ([Temporal][8])

---

# 8) End-to-end example (SQL MI, Active-Passive between regions)

1. **Provisioning (Terraform):**

   * MI in Region A (zone-redundant).
   * MI in Region B (zone-redundant).
   * **Failover Group** across A↔B.
   * AKV secret template (just the **name**; value supplied at runtime). ([Microsoft Learn][5])
2. **App binding:** VM tier reads **`DB_CONN_URI`** from AKV at runtime.
3. **Runbook: Planned switchover (Temporal):** drain → FOG planned failover → AKV rotation → warm-up → resume.
4. **Runbook: Unplanned failover:** detect → force failover → AKV rotate → app verification → incident workflow.
5. **Runbook: Failback:** re-seed, planned switchover back, reconcile state.

No VM apply was needed in any of those Day-2 operations.

---

# 9) End-to-end example (PG Flex, Geo-Replica DR)

1. **Provisioning (Terraform):**

   * PG Flex in Region A with **HA** (zone-redundant).
   * **Geo-replica** in Region B. ([Microsoft Learn][1])
   * AKV secret template (`PG_URI`).
2. **Runbook: Switchover (Temporal):** drain writes → check lag ≈ 0 → **promote** replica in B → rotate AKV → warm-up → resume.
3. **Runbook: Region loss:** PITR restore in B → rotate AKV → resume → later re-establish replica to A. ([Microsoft Learn][3])

---

# 10) Governance, safety, and UX

* **Approvals:** Temporal adds manual approval gates for switchover/failback in prod.
* **Blast-radius control:** runbooks operate only on the **DB tier** + **AKV value**, never the VM tier.
* **Observability:** emit events (step started/succeeded), Terraform-free; stream to UI and store history.
* **Testing:** schedule **DR drills** (Temporal cron) quarterly: non-prod promotion/restore tests with automatic teardown.
* **Security:** RBAC by Entra groups on runbooks (who can force failover), scoped MI/PG identities, AKV RBAC for secret updates only.

---

# 11) Recommended path

1. **Encode HA/DR topology in Terraform** per tier (enable MI zone redundancy + FOG; PG Flex HA + replicas; AKV backups). ([Microsoft Learn][1])
2. **Build Temporal runbooks** for: planned switchover, unplanned failover, failback, PITR restore, DR drills.
3. **Bind apps to AKV only** (one secret to rule them all); runbooks only change secret **value**.
4. **Add drift jobs** (plan-only) + alerting; add post-event “state reconciliation” tasks.
5. **Document RTO/RPO** per app; choose Pattern B for most production workloads.

This split—**Terraform for desired topology**, **Temporal for Day-2 buttons**—lets you achieve real HA/DR without the risk of a broad “terraform apply” across tiers during an incident, while keeping all actions auditable and repeatable.

---

**References**
PG Flex HA, zones & constraints; geo-replicas & cross-region promotion; backups & PITR. ([Microsoft Learn][1])
SQL MI failover groups (cross-region DR); MI zone redundancy. ([Microsoft Learn][5])
AKV DR/backup guidance. ([Microsoft Learn][6])
Terraform resources (MI failover group, PG Flex). ([Terraform Registry][7])
Temporal durable execution & case studies (long-running DB operations, retries). ([Temporal][8])

If you want, I can sketch the **Temporal workflow interfaces** (signals/queries, activity contracts) for these runbooks next, or produce a **Day-2 CLI** spec (e.g., `smartinfra db switchover --tier sqlmi --env prod`).

[1]: https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/how-to-configure-high-availability?utm_source=chatgpt.com "Configure high availability - Azure Database for PostgreSQL"
[2]: https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/concepts-read-replicas?utm_source=chatgpt.com "Read replicas - Azure Database for PostgreSQL"
[3]: https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/overview?utm_source=chatgpt.com "What Is Azure Database for PostgreSQL flexible server?"
[4]: https://learn.microsoft.com/en-us/azure/azure-sql/managed-instance/high-availability-sla-local-zone-redundancy?view=azuresql&utm_source=chatgpt.com "Availability Through Local and Zone Redundancy - Azure ..."
[5]: https://learn.microsoft.com/en-us/azure/azure-sql/managed-instance/failover-group-sql-mi?view=azuresql&utm_source=chatgpt.com "Failover groups overview & best practices - Azure SQL ..."
[6]: https://learn.microsoft.com/en-us/azure/key-vault/general/disaster-recovery-guidance?utm_source=chatgpt.com "Azure Key Vault availability and redundancy"
[7]: https://registry.terraform.io/providers/hashicorp/azurerm/3.104.1/docs/resources/mssql_managed_instance_failover_group?utm_source=chatgpt.com "azurerm_mssql_managed_insta..."
[8]: https://temporal.io/blog/temporal-replaces-state-machines-for-distributed-applications?utm_source=chatgpt.com "Temporal: Beyond State Machines for Reliable Distributed ..."
[9]: https://medium.com/%40surajsub_68985/workflow-orchestration-meets-infrastructure-as-code-5ce79a186ae2?utm_source=chatgpt.com "Workflow Orchestration meets Infrastructure as Code"
[10]: https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/concepts-read-replicas-geo?utm_source=chatgpt.com "Geo-replication - Azure Database for PostgreSQL"
[11]: https://learn.microsoft.com/en-us/azure/storage/common/storage-disaster-recovery-guidance?utm_source=chatgpt.com "Azure storage disaster recovery planning and failover"
