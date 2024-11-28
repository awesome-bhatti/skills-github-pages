---
title: "Cloudflare Logs Incident: November 14, 2024"
date: 2024-11-28
---


# Cloudflare Logs Incident: November 14, 2024

On November 14, 2024, Cloudflare experienced an incident that impacted the majority of customers using Cloudflare Logs. During the 3.5 hours of service disruption, approximately 55% of the logs were not sent and were lost. We deeply regret this issue and are committed to preventing similar occurrences.

This post explains:
- What happened
- Steps we’re taking to prevent recurrences
- Insights for engineering teams about the systems and failure class involved

---

## Importance of Subsystem Protection

Failures at scale are inevitable. Subsystems must protect themselves from failures in other parts of the system to prevent cascading issues. In this case:
- A misconfiguration in one subsystem caused a cascading overload in another.
- Proper configuration could have mitigated the issue and prevented the loss of logs.

---

## Background

Cloudflare’s global network generates event logs for various services. These logs contain metadata about system operations, such as every request to Cloudflare’s CDN. Cloudflare Logs makes these event logs available to customers for:
- Compliance
- Observability
- Accounting

### Statistics
- Cloudflare sends about **4.5 trillion event logs daily** to customers.
- These logs represent less than 10% of the **50 trillion total event logs processed**.

---

## System Architecture

### Overview
Cloudflare’s network comprises:
- **Tens of thousands of servers and network hardware**
- **Specialized software programs** across 330+ cities globally

Event logs are pushed to customers primarily through **Logpush**, a service that aggregates logs into manageable file sizes.

### Key Components
#### **Logfwdr**
- Written in Golang
- Forwards event logs in batches to **Logreceiver**
- Determines where logs should be sent based on configuration

#### **Logreceiver**
- Written in Golang
- Sorts and demultiplexes event logs into customer-specific batches
- Handles about **45 PB of customer event logs daily**

#### **Buftee**
- Internal buffering system written in Golang
- Provides named buffers for data management
- Handles over **1 million buffers globally**
- Prevents "head of line" blocking and supports multi-consumer data pipelines

#### **Logpush**
- Reads logs from Buftee buffers and pushes them to customer destinations
- Processes over **600 million batches daily**

---

## What Happened

### Timeline of Events
1. **Change Introduced**
   - A change was made to support a new dataset for Logpush.
   - A bug caused a blank configuration to be sent to **Logfwdr**, resulting in no logs being forwarded.

2. **Failsafe Triggered**
   - The “fail open” mechanism in Logfwdr sent logs for all customers, not just those configured.
   - This caused a massive spike in logs and overwhelmed downstream systems.

3. **Buftee Overload**
   - Buftee began creating buffers for all customers, leading to:
     - A 40x increase in buffers.
     - Buftee managing **40 million buffers globally**.

4. **System Recovery**
   - The overload required a complete system reset and restart.

---

## Root Causes

1. **Bug in Logfwdr Configuration System**
   - This bug was likely to occur, but the broader system was not tested for a fail-open scenario.

2. **Failure of Buftee Safeguards**
   - Mechanisms to handle buffer overloads were not properly configured.
   - Buftee, intended as a safeguard, became unresponsive under the increased load.

### Analogy
Failing to configure Buftee safeguards is like having a seatbelt in a car but not fastening it. The safeguard was in place but could not perform its function when needed.

---

## Going Forward

1. **Improved Alerting**
   - New alerts to detect misconfigurations early.

2. **Enhanced Testing**
   - Regular “overload tests” to simulate similar cascades.
   - Continued “cut tests” to assess system resilience under datacenter or network loss.

3. **System Hardening**
   - Fixing specific bugs and testing safeguards to ensure robustness.

### Vision for Logpush
Logpush remains a robust platform for customer log integration. Its flexibility allows:
- Multiple destinations for logs
- Filtering and customization to suit customer needs
