---
Author: Joongi Kim (joongi@lablup.com)
Status: Draft
Created: 2025-11-16
---

# Preemption of Low-priority Sessions

## Motivation

Currently we have a priority number in the range of 0..100 integer, whereas the default value is 10.
The scheduler schedules the pending sessions with the highest priority value first.
If there are still remaining resources, then it schedules the pending sessions with the next-highest priority value.

There are some use cases that we need to preempt already-running low-priority sessions to start the pending sessions with higher priority as quickliest as possible.

## Specification

### Configuration

| Field | Scope | Default | Description |
|-------|-------|---------|-------------|
| `preemptible_priority` | Resource Group | 5 | The priority number threshold to make running sessions with the priority value lower or equivalent to this preemptible. |
| `preemption_order` | Resource Group | "oldest" | Preempt "oldest" or "newest" running sessions first when they have same (lower) priority. |

Since the scheduler runs in the resource group scope, this configuration should be per resource group.

### Implementation

The scheduler determines if it needs to perform preemption as follows:

- There are not enough resources to schedule a pending session.
- There are running sessions with lower priority than the pending session in the front of the job queue.

When doing preemption, there may be multiple running sessions with same or different priority values to preempt.
Choose the preempted sessions by the following order, until we have enough released resources to start the pending session:

* Lowest priority value
* The job start time
  - Oldest first or newest first depending on `preemption_order` configuration

### Slurm-like Interface

Slurm has a concept of partitions.
If a job is provisioned inside "preemptible" partitions can be terminated by the scheduler when there are resource competition.

The preemptible partition could be simulated by setting the priority value of pending sessions and adjusting `preemptible_priority`.

### Automatic resumption of preempted sessions

> [!WARNING]
> We need discussion on this section.

Currently there is no automatic resumption once a running session is terminated.

However, this may be required for cases like:

- The preempted session is a batch job, which has internal checkpointing and resumption mechanisms.

Though, it may not be required when:

- The preempted session is an inference job, which has no auto-scaling configuration applied.
- The preempted session is an interactive job, which needs human intervention to resume work.

If we want to resume the preempted session, we need to keep its configuration and metadata when terminated,
and make it a pending session again when there are released resources.
We need to define the conditions and how to achieve this.
