# nri-autodiscover-config-rds

Auto-discovery of endpoints for monitoring with the `nri-postgresql` plugin using the New Relic NerdGraph API to filter instances from the Infrastructure RDS integration.

- [Dynamic RDS Discovery & Capability Probing](#dynamic-rds-discovery-capability-probing)
- [Database Availability & Performance Monitoring](#database-availability-performance-monitoring)
- [Global Cloud Inventory & RDS Events](#global-cloud-inventory-rds-events)
- [APM Pixie for DB](#apm-pixie-for-db)
- [PostgreSQL and Valkey Transparent Proxy](#postgresql-and-valkey-transparent-proxy)

---

## Dynamic RDS Discovery & Capability Probing

### Background
The New Relic Infrastructure Agent manages off-host integrations through a modular system supporting variable replacement and dynamic `register_config` actions. Currently, `nri-postgresql` requires manual configuration for each database host.

### Proposal
Create a lightweight discovery integration, `nri-rds-discovery`, to automate the link between high-level cloud metrics and deep database-level insights.
- **Infrastructure Discovery:** Query NerdGraph for active `RDS_INSTANCE` or `AURORA_INSTANCE` entities.
- **Capability Probing:** Optionally connect to discovered instances to identify extensions (e.g., `pg_stat_statements`), enumerate databases, and tune monitoring configurations.
- **Availability Tracking:** Support `INCLUDE_ON_FAILURE` flags to ensure instances are monitored for availability even when metric collection is blocked by authentication or network issues.

### Roadmap
- [ ] **Prototype NerdGraph Query:** Fetch RDS/Aurora instances with connection endpoints.
- [ ] **Design Discovery Integration:** Develop the Go-based `nri-rds-discovery` tool.
- [ ] **Implement Config Protocol:** Output the `register_config` JSON payload for the agent.
- [ ] **End-to-End Validation:** Verify the agent successfully spawns workers based on discovery.

---

## Database Availability & Performance Monitoring

### Proposal
Enhance `nri-postgresql` to provide deep, built-in availability and performance metrics during every collection cycle, eliminating the need for external heartbeat services.

### Key Capabilities
- **Response Time Breakdown:** Decompose query duration into DNS Lookup, TCP Connection, TLS Handshake, Authentication, and Time to First Byte (TTFB).
- **Implicit Health Checks:** Report throughput and exact error details (e.g., auth failure vs. timeout) for every internal collection query.
- **Connection Analysis:** Identify backend hostnames, Read/Write status, and performance deltas between "cold" (new) and "warm" (cached) connections.

### Configuration
- `COLLECT_AVAILABILITY_METRICS`: Enable/disable the availability suite.
- `AVAILABILITY_QUERY`: Custom SQL for health checks (defaults to `SELECT 1`).
- `USE_CONNECTION_POOL`: Toggle persistent connections for performance comparison.

---

## Global Cloud Inventory & RDS Events

### Background
Customers with many accounts struggle with "account drift" (unmonitored databases) and missing lifecycle context (reboots, maintenance, failovers) that impacts database performance.

### Proposal
Deploy a singleton remote integration, `nri-cloud-inventory`, to provide a global view of the database fleet and capture infrastructure-level events.
- **Cross-Account Scanning:** Use AWS Organizations/AssumeRole to scan all accounts and report hardware sizing, regions, and cluster topologies to New Relic Inventory.
- **Event Forwarding:** Poll `DescribeEvents` or subscribe to EventBridge to translate RDS lifecycle events into actionable New Relic Infrastructure Events.

### Benefits
- **Drift Detection:** Instantly identify gaps where RDS instances exist in AWS but lack deep monitoring.
- **Event Correlation:** Overlay maintenance or failover events on performance charts to identify root causes.

---

## APM Pixie for DB

### Proposal
Leverage Pixie on Kubernetes to auto-configure application-side database monitoring, providing observability from the perspective of the application client.

### Key Capabilities
- **Query Fingerprinting:** Automatically parse SQL traffic to map databases, tables, users, and operation types (INSERT, UPDATE, DELETE).
- **Automated Explain Plans:** Capture production query parameters and generate Explain Plans for performance tuning without manual intervention.

---

## PostgreSQL and Valkey Transparent Proxy

### Proposal
Implement or integrate an observability-focused proxy layer (using HAProxy, ProxySQL, Envoy, or a custom Go-based solution) to gain centralized control and visibility over all database traffic.

### Key Use Cases & Benefits
- **Performance Testing:** Use AWS Aurora snapshots and Kafka to record and replay production workloads against isolated test environments.
- **Platform-Level Validation:** Provide an opt-in service for teams to validate major version upgrades against real traffic on disposable "throw-away" backends.
- **Reliability:** Centralized connection pooling, automatic retries, and graceful handling of DNS caching or failover events.
- **Migration & Upgrades:** Enable seamless blue/green transitions and dual-writing during large-scale architectural shifts.
- **Traffic Control:** Centralized query management, including inline query killing and queuing to buffer brief backlogs.
