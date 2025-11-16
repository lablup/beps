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

| Field | Scope | Description |
|-------|-------|-------------|
| `preemptible_priority` | Resource Group | The priority number threshold to make running sessions with the priority value lower or equivalent to this preemptible. |

Since the scheduler runs in the resource group scope, this configuration should be per resource group.

### Slurm-like Interface

Slurm has a concept of partitions.
If a job is provisioned inside "preemptible" partitions can be terminated by the scheduler when there are resource competition.

### Automatic restoration of preempted sessions

> [!WARNING]
> We need discussion on this section.

Currently there is no automatic restoration once a running session is terminated.

However, this may be required for cases like:

- The preempted session is a batch job, which has internal checkpointing and resumption mechanisms.

Though, it may not be required when:

- The preempted session is an inference job, which has no auto-scaling configuration applied.
- The preempted session is an interactive job, which needs human intervention to resume work.
