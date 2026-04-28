# Shella Lolitha

Database Administrator based in Jakarta, Indonesia. I work at the intersection of database operations and infrastructure automation — keeping databases reliable at scale while building the tooling that makes that reliability sustainable.

[LinkedIn](https://linkedin.com/in/shellalolitha) · shellalolitha@gmail.com

---

## How I work

Most database problems are not database problems. They're visibility problems, process problems, or scaling problems that happen to surface in the database layer. My instinct is always to understand the system before touching it — what's actually happening, why, and what the second-order effects of a change would be.

I'm drawn to work that sits at the edge of operations and engineering: designing systems that make the right thing the easy thing, so that reliability doesn't depend on someone remembering to do something manually.

The projects below are examples of that kind of work.

---

## Projects

### RDS Capacity Planning System
`AWS CloudWatch` `PostgreSQL` `n8n` `Metabase`

#### The problem

Instance sizing decisions across a multi-environment AWS RDS fleet were made reactively — someone noticed a slow query, or a CloudWatch alarm fired, and a ticket got opened. There was no systematic view of whether instances were correctly sized, no early warning for instances trending toward saturation, and no shared baseline for what "well-utilized" even meant.

The result was a mix of over-provisioned instances (wasteful) and under-provisioned ones (risky), with no easy way to tell which was which.

#### The approach

I built a capacity planning pipeline that pulls CloudWatch CPU metrics — specifically tm99 and max — and processes them through a set of PostgreSQL materialized views designed to answer different questions at different time horizons.

The four views:

- `tm99_2m` — rolling 2-month tm99 baseline per instance. This is the primary signal for steady-state utilization.
- `spike_7d` — 7-day spike detection. Separates instances with genuinely high load from those with occasional bursts that don't represent real capacity pressure.
- `recommendation` — computes a target vCPU using `current_vcpu × (avg_tm99 / 40)`, then normalizes to the nearest standard AWS instance tier. The 40% target gives headroom for spikes without leaving capacity on the table.
- `lifecycle` — tracks resizing history and flags instances where utilization is consistently low enough to be candidates for Aurora Serverless v2.

n8n orchestrates the refresh cycle and pushes results into Metabase dashboards, which become the single source of truth for capacity review conversations.

#### Why this design

I chose materialized views over live queries because the source data doesn't need to be real-time — capacity decisions are made on trend, not on the last five minutes. Materialized views also let each layer of analysis be reasoned about independently, which made the logic easier to validate and easier to change.

The 40% tm99 target was deliberately conservative. RDS CPU behavior is nonlinear near saturation — headroom matters more than it looks like it does on a dashboard.

---

### DBA Automation Workflows
`n8n` `Python` `Bytebase` `Jira` `AWS CloudWatch` `pgaudit` `PostgreSQL`

Repetitive operational work is the enemy of good database engineering — it crowds out the thinking time that actually improves the system. I've built a set of automation workflows at Kredivo to handle the recurring tasks, so the team can stay focused on work that requires judgment.

- **Audit log analysis** — Queries CloudWatch Logs Insights against pgaudit SESSION logs on a schedule, parses entries with regex-based field extraction, and produces structured summaries by user, object, and statement type. Shifted audit from a reactive exercise to a routine one.
- **Bytebase access approval** — Automates the approval routing for database access requests in Bytebase. Requests are validated against access policy, routed for sign-off where required, and provisioned on approval — with a full audit trail by default.
- **Grant access automation** — Handles role and permission grants across environments without manual DBA intervention. Triggered by approved requests, executed consistently, logged automatically.
- **Create user across all platforms** — Automates user creation across all database platforms (PostgreSQL, MySQL, across AWS and GCP environments) from a single trigger. Eliminates the multi-step, error-prone manual process of provisioning across environments one by one.
- **Bloated index detection → Jira → reindex pipeline** — Detects bloated indexes automatically, opens a Jira ticket, and notifies the owning engineer. Before execution, the workflow checks current database conditions (replication lag, active connections, lock state) and leaves the findings as a Jira comment. Once the ticket is moved to the appropriate status, the reindex is executed via a Jenkins CI/CD pipeline — so the engineer stays in the loop, but the DBA doesn't need to be in the room.

---

### PostgreSQL → MySQL Migration Framework
`PostgreSQL` `MySQL` `GCP CloudSQL` `Kafka` `Debezium` `Golang` `Bash` `Docker` `Ansible`

*This was a team effort. The framework and architecture were designed and built collectively — I contributed as one of the engineers on the project, with a focus on the replication pipeline and automation tooling.*

#### The problem

Tokopedia ran PostgreSQL as its primary database engine, hosted on VMs. After the ByteDance acquisition, the directive was to migrate everything to ByteRDS — ByteDance's in-house managed MySQL cloud. This was not a simple lift-and-shift. It was three migrations in one:

- **Cloud provider migration** — from our own VM-based infrastructure to ByteRDS
- **Database engine migration** — from PostgreSQL to MySQL
- **Deployment model migration** — from self-managed VMs to a fully managed database service

200+ services. Terabytes of data. Near-zero downtime required.

#### The approach

The team designed a staged migration pipeline to manage the complexity of moving this much data across engine and infrastructure boundaries simultaneously.

**Schema transformation.** Before any data moved, PostgreSQL DDL had to be converted to MySQL-compatible DDL — type mapping, constraint translation, index equivalency. This was automated, but required careful judgment calls: PostgreSQL and MySQL don't map cleanly in all cases (sequences vs. auto-increment, array types, partial indexes), and those decisions had to be consistent across every service.

**The replication topology.** ByteRDS is a fully managed service — we had no superuser access, which ruled out setting it up directly as a replication target. To work around this, the team introduced GCP CloudSQL (MySQL) as a middleware layer. CloudSQL was configured as the replication master; ByteRDS was set up as its replica. This meant we never had to touch ByteRDS directly during the data load phase — it stayed in sync passively through standard MySQL replication.

**CDC pipeline.** Debezium captured row-level changes from the source PostgreSQL VM via CDC. Kafka transported the events. A consumer applied them to CloudSQL in near real-time, with ByteRDS trailing behind as a replica. Loading terabytes while keeping pace with live writes — and matching data types, indexes, and constraints correctly across engine boundaries — was the hardest part of the project.

**My contribution — pipeline automation helpers.** Within the replication pipeline, I built a set of automation tools to reduce manual intervention during migrations:
- Auto-connect Kafka connector — handles connector registration and retry logic without manual steps
- Replication slot checker — monitors whether replication slots are active and alerts on false/stale states before they cause lag to accumulate
- Auto-deploy connector via Jenkins — integrates connector deployment into the CI/CD pipeline so rollouts are consistent and auditable

**Cutover.** Once replication lag was within the agreed threshold, cutover began: application connections were cut from PostgreSQL and pointed to ByteRDS. ByteRDS was then promoted from replica to master. CloudSQL was decommissioned. The rollback path was defined before every cutover — if promotion failed, connections could be redirected back to the PostgreSQL source.

#### The tradeoffs

The middleware topology (PostgreSQL → CloudSQL → ByteRDS) added a hop and introduced two potential failure points instead of one. That was a deliberate tradeoff — it was the only way to load data into ByteRDS without superuser access, and the replica lag between CloudSQL and ByteRDS was consistently small enough to be acceptable.

Agreeing on a cutover lag threshold upfront was also non-trivial. "In sync" had to mean something specific and measurable, not just "looks about right." Getting that alignment before migration day mattered.

**Outcome:** 200+ services migrated with near-zero downtime. Per-service migration effort reduced by ~70% compared to manual approaches. The framework was reused across teams without requiring deep migration expertise on the service owner side.

---

*All projects involve proprietary systems. This portfolio documents architecture, design decisions, and technical reasoning — not production code.*
