# nri-autodiscover-config-rds

Auto discovery the endpoints to monitor with the nri-postgresql plugin using New Relic NerdGraph API to filter instances coming from Infrastructure RDS integration for CloudWatch metrics 

Brainstorming improvements with Gemini...

# New Relic Infrastructure: Dynamic Integration Discovery

## Goal
Automate the configuration of `nri-postgresql` for AWS RDS/Aurora instances by leveraging existing New Relic data. Instead of manual YAML configuration for every database host, we aim to use the New Relic API (NerdGraph) to discover reporting RDS entities and dynamically instantiate the PostgreSQL integration.

---

## Current Configuration Architecture

The New Relic Infrastructure Agent manages off-host integrations through a modular system that supports both static and dynamic configurations.

### 1. Databind & Discovery
The `databind` package is the core of the agent's dynamic configuration system. It allows for variable replacement (e.g., secrets from Vault/AWS KMS) and basic discovery (e.g., Docker/Fargate containers).
- **Relevant Files:**
    - [`infrastructure-agent/pkg/databind/pkg/databind/config.go`](infrastructure-agent/pkg/databind/pkg/databind/config.go): Defines `Discovery` sources and `Variables`.
    - [`infrastructure-agent/pkg/databind/internal/discovery/command/command.go`](infrastructure-agent/pkg/databind/internal/discovery/command/command.go): Implements `command` discovery, which can execute a script to return targets.

### 2. Config Protocol V1 (`register_config`)
The most powerful mechanism for dynamic configuration is the **Config Protocol**. An integration can output a specific JSON payload to `stdout`, which the agent then uses to register and manage other integrations.
- **Action:** `register_config`
- **Protocol Definition:** [`infrastructure-agent/pkg/integrations/configrequest/protocol/v1.go`](infrastructure-agent/pkg/integrations/configrequest/protocol/v1.go)
- **Workflow:** A "discovery integration" runs, queries an external source (like NerdGraph), and outputs a list of integration configurations for the agent to execute.

---

## `nri-postgresql` Integration

The PostgreSQL integration is a standalone binary that collects metrics and inventory from PostgreSQL instances.
- **Main Entry Point:** [`nri-postgresql/src/main.go`](nri-postgresql/src/main.go)
- **Configuration:** It relies on environment variables or CLI arguments defined in [`nri-postgresql/src/args/argument_list.go`](nri-postgresql/src/args/argument_list.go).
- **Key Parameters for RDS:**
    - `HOSTNAME`: The RDS endpoint.
    - `USERNAME` / `PASSWORD`: Credentials (can be sourced via `databind` variables).
    - `IS_RDS`: Set to `true` to enable RDS-specific metric collection.

---

## Proposal: `nri-rds-discovery` & Intelligent Capability Discovery

We propose creating a new, lightweight discovery integration that automates the link between AWS CloudWatch/Kinesis metrics and the deep database-level metrics provided by `nri-postgresql`. 

### 1. Infrastructure Discovery
- **Query NerdGraph:** The tool queries New Relic's NerdGraph API for entities of type `RDS_INSTANCE` or `AURORA_INSTANCE` that are already reporting.
- **Filtering:** Users can provide filters (e.g., tags or regions) to limit which instances are targeted.

### 2. Intelligent Capability Discovery
Once an endpoint is discovered, the plugin can optionally connect to the instance to auto-discover its capabilities and dynamically tune the `nri-postgresql` configuration payload:
- **Database Enumeration:** Automatically determine which databases exist and should be monitored (populating `COLLECTION_LIST`).
- **Extension Probing:** Check for `pg_stat_statements` to automatically enable `ENABLE_QUERY_MONITORING`, or check for `tablefunc` to enable `COLLECT_DB_LOCK_METRICS`.
- **Opt-out Support:** Provide an `AUTO_DISCOVER_CAPABILITIES` toggle so users can disable probing and rely on strict YAML overrides.

### 3. Smart Failure Handling
During capability discovery, connections might fail. The tool will support configurable inclusion criteria to decide whether to still register the integration:
- `INCLUDE_ON_AUTH_FAILURE`: If capability probing fails due to bad passwords, still register the integration so `nri-postgresql` can continuously report the Authentication Failure as an availability metric.
- `INCLUDE_ON_NETWORK_FAILURE`: Register the integration even if the initial probe times out, allowing the main integration to track network availability over time.

### 4. Registration & Lifecycle
- For each valid match (based on discovery and failure handling rules), the tool generates an `nri-postgresql` configuration entry and sends it to the Infrastructure Agent using the `register_config` action.
- The agent manages the execution of these instances, ensuring they stay up-to-date with the environment.

---

## Feature Proposal: Database Availability & Performance Monitoring

We aim to enhance `nri-postgresql` to provide built-in, deep availability and performance metrics for every collection cycle. This reduces the need for external "heartbeat" applications.

### Key Capabilities:
1. **Implicit Availability:** Every metrics collection cycle should report:
    - **Response Time:** Duration of each query executed by the plugin.
    - **Throughput:** Success/failure rates of internal collection queries.
    - **Error Details:** Capture exact error messages, response output, and error codes (e.g., authentication failures vs. timeouts).
2. **Configurable Heartbeat:**
    - `ENABLE_AVAILABILITY_QUERY`: Toggle a dedicated "heartbeat" query (e.g., `SELECT 1`).
    - `AVAILABILITY_QUERY`: Allow users to override the heartbeat query with custom logic.
3. **Granular Connection Timing:**
    To provide deep visibility into *why* a connection might be slow or failing, we will break down the "Response Time" into its constituent phases:
    - **DNS Lookup Time:** Time taken to resolve the database hostname.
    - **TCP Connection Time:** Time to establish the base network socket.
    - **TLS Handshake Time:** Duration of the SSL/TLS negotiation (critical for RDS/Aurora).
    - **Authentication Time:** Time spent on the PostgreSQL wire protocol login/handshake.
    - **Time to First Byte (TTFB):** Duration from query send to the start of the response.
    - **Full Query Response Time:** The end-to-end execution time for the heartbeat or collection query.

4. **Connection State Tracking:**
    - **Backend Resolution:** Identify the specific backend hostname of the connection.
    - **Read/Write Status:** Detect if the instance is a writer or a read-only replica.
4. **Connection Strategy Analysis:**
    - **New vs. Cached Connections:** Support both opening/closing a connection per interval AND maintaining a single-connection pool.
    - **Performance Delta:** Report the difference in response time between "cold" (new) and "warm" (cached) connections.

### Proposed Configuration Flags:
- `COLLECT_AVAILABILITY_METRICS` (bool): Enable/disable the entire availability suite.
- `AVAILABILITY_QUERY` (string): The SQL to run for health checks (default: `SELECT 1`).
- `USE_CONNECTION_POOL` (bool): Whether to maintain a persistent connection for performance comparison.

---

## Feature Proposal: Global Cloud Inventory & RDS Events

To bridge the gap between specific host-based database metrics and high-level cloud architecture, we propose integrating a global scanning service into the New Relic ecosystem, designed to capture cross-account AWS/Azure inventory and RDS lifecycle events.

### The Problem
Currently, New Relic relies on the AWS CloudWatch integration (often via Kinesis metric/log streams) or the host-level Infrastructure Agent for data. However:
1. **Account Drift:** Customers with dozens of AWS/Azure accounts struggle to ensure all accounts are integrated with New Relic.
2. **Missing Lifecycle Events:** Critical RDS events (reboots, parameter group changes, upcoming maintenance, failovers) are not natively captured as actionable events in New Relic.
3. **Inventory Context:** Customers need a global, read-only view of their database fleets (hardware sizing, region, cluster topology) to correlate with `nri-postgresql` performance data.

### Proposal: `nri-cloud-inventory` (Remote Integration)

Rather than forcing the local host-agent to do global discovery, we propose a new **Remote Integration** (similar to your existing `nr-inventory` service) that customers can deploy as a singleton (e.g., as a container in ECS/Fargate or a Lambda function).

#### 1. Cross-Account Inventory Polling
- **Mechanism:** The service uses AWS Organizations/AssumeRole to scan across all known child accounts.
- **Data Payload:** It uses the New Relic Integrations SDK's `SetInventoryItem` to report database metadata (e.g., `db.instance.class`, `db.engine.version`, `db.cluster.members`).
- **Destination:** This data lands in New Relic's `InfrastructureEvent` or `Inventory` types, providing a definitive, continually updated CMDB of all databases, *even those not currently monitored by nri-postgresql*.

#### 2. RDS Event Capture & Forwarding
- **Mechanism:** The service polls the AWS `DescribeEvents` API (or subscribes to an AWS EventBridge/SNS topic) for RDS and Aurora events.
- **Data Payload:** It translates these AWS events into New Relic Infrastructure Events (e.g., `EventType: RdsLifecycleEvent`, `Action: maintenance-started`).
- **Terraform Automation:** Provide a standardized Terraform module that customers can use to automatically route AWS EventBridge rules for RDS into New Relic's Log API or directly to this new service.

### Value to the Customer
- **Single Pane of Glass:** Combines the deep query performance of `nri-postgresql` with the broad architectural context of the cloud provider.
- **Drift Detection:** Dashboards can highlight RDS instances that exist in AWS Inventory but have no corresponding `nri-postgresql` metrics, instantly identifying monitoring gaps.
- **Event Correlation:** When a spike in response time occurs in `nri-postgresql`, users can overlay `RdsLifecycleEvents` to see if a background backup or failover was the root cause.

1. **[ ] Prototype NerdGraph Query:** Develop a GraphQL query to fetch RDS/Aurora instances with their connection endpoints and relevant metadata.
2. **[ ] Design Discovery Integration:** Create a small Go-based tool (e.g., `nri-rds-discovery`) that implements this query.
3. **[ ] Implement Config Protocol Output:** Ensure the tool can output the `register_config` JSON payload required by the Infrastructure Agent.
4. **[ ] Availability POC:** Update `nri-postgresql` connection logic to track DNS lookup and query execution times.
5. **[ ] End-to-End Validation:** Verify that the agent successfully spawns `nri-postgresql` workers based on the discovery output.


## APM Pixie for DB
Explore auto-configuring APM on K8s with Pixie for application-side database monitoring.
- Parse and fingerprint SQL queries to map databases, tables, users, and operation types (INSERT, UPDATE, DELETE).
- Capture query parameters and automatically generate Explain Plans.

## PostgreSQL and Valkey Transparent Proxy
Explore using open-source solutions (HAProxy, ProxySQL, Envoy) or developing an in-house Go-based proxy to enable query-level observability via New Relic APM/Insights.

**Key Use Cases & Benefits:**
- **Performance Testing & Workload Simulation:** Use AWS Aurora snapshots combined with Kafka to record and replay production workloads. The proxy layer captures live operations, streams them to Kafka, and then consumes that stream to run realistic performance tests against isolated database copies.
- **Observability:** Centralized query monitoring and fingerprinting to map databases, tables, and operation types.
- **Reliability:** Connection pooling to reduce server load, automatic retries, and graceful handling of failover/DNS caching issues.
- **Migration & Upgrades:** Support for major version upgrades and significant architectural changes via traffic mirroring, dual-writing, and live data synchronization.
- **Platform-Level Validation (Opt-in):** A centralized, opt-in service for teams to validate new database versions against real production traffic on hidden, disposable backends. This removes the burden on individual teams to build custom validation logic for upgrades, allowing for "throw-away" testing before true promotion.
- **Deployment Flexibility:** Localized proxy endpoints within K8s clusters to abstract backend infrastructure and enable cross-cloud/cluster mobility.
- **Traffic Control:** Centralized query management (e.g., query killing/timeouts) and queuing to buffer brief backlogs or error events.




