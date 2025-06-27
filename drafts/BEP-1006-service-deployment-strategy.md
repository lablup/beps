---
Author: Jeongseok Kang (jskang@lablup.com)
Status: Draft
Created: 2025-06-27
---

# Backend.AI Model Service Deployment Strategy Enhancement

## Abstract

This proposal aims to enhance the Backend.AI Model Service deployment process by providing more sophisticated, zero- or near-zero-downtime strategies. Specifically, it outlines how to implement rolling updates and optional blue-green deployments to ensure minimal service interruption. In addition, it proposes a way to dynamically update environment variables and to rollback when needed.

## Motivation

As model serving usage has been growing, Backend.AI introduced the “Model Service” in version 23.09 and Backend.AI FastTrack introduced the “Model Serving Task” in 25.09 to meet evolving market needs. However, the current deployment process does not fully support strategies for zero-downtime deployment—an important requirement for end user-facing services. To address this limitation, we propose:
• A rolling update mechanism that updates services incrementally while keeping a subset of instances operational.
• A blue-green deployment flow suitable for use cases requiring isolated testing or strict regulatory requirements.
• A rollback capability that preserves service continuity in case of errors in the new version.

## Design

### AS-IS

- Deployments only support simple replacement or upgrades, causing short service interruptions.
- Environment variables are fixed at deployment time and cannot be dynamically updated.
- The system lacks fine-grained versioning (e.g., a dedicated “version” field in the `Routing` object) for easier rollback or co-existence of multiple versions.

### TO-BE

1. Introduce a Rolling Update Strategy:
- When a user triggers a promotion or update, Backend.AI attempts to create new model service sessions incrementally.
- If there are insufficient resources (e.g., “replicas × resource-required” is more than what is available), the update request is immediately rejected.
- New instances are started and tested for health. As soon as a new instance is deemed healthy, a proportion of traffic is shifted to it. This continues until all instances are running the new version.
- If any instance fails during rollout, the system can automatically or manually roll back to the last stable version.

2. Introduce a Blue-Green Deployment Mode:
- A user initiates a deployment update via API (UI and CLI support planned).
- The existing active deployment is referred to as “Blue.” The new one is “Green.”
- The system creates new “Green” routings—each with an initial traffic ratio of 0.0—equal to the desired replica count.
- Once the new “Green” routings are all healthy (or have passed custom validation checks), the system updates `traffic_ratio` to 1.0 for Green and to 0.0 for the old Blue.
- Blue routings are retained temporarily for rollback purposes or removed entirely if deemed stable.
- The Model Service returns to a `HEALTHY` (or “active”) status.

3. Enable Updating of Environment Variables:
- Allow environment variables (envs) to be modified during rolling or blue-green updates, removing the need for redeployment solely for env changes.
- Provide graceful handling of env changes so that sessions have time to reload configurations and verify correctness.

4. Enhance Versioning:
- Each `Routing` gains a new field: `version`.
- The system can track which version is currently in production (and previous stable ones), making rollbacks simpler.

### Detailed Flow for Rolling Updates

#### Rolling Update

1. User triggers or submits a promotion.
2. If insufficient resources exist to accommodate additional replicas, the request is rejected.
3. The system launches a new instance (linked to the updated version).
4. Once the instance is healthy, routing adjusts to direct a small percentage of traffic to it.
5. Gradually scale up traffic to the new version.
6. After all instances are updated and validated, the previous version can be terminated or kept temporarily for rollback.

#### Blue-Green Update

1. User triggers a model service update (API/UI/CLI).
2. Service transitions to `UPDATING` state.
3. The system creates the new set of routing objects (“Green”) with `traffic_ratio=0.01.
4. As soon as all Green routes are `HEALTHY`, further validations (e.g., canary requests, manual checks) can be conducted.
  - Optionally, the user can press a “Confirm Deploy” button once they verify correct behavior.
5. Update the Green routing `traffic_ratio` to 1.0, reduce Blue to 0.0.
6. Optionally keep Blue routing briefly for quick rollback, or remove it and free resources.
7. Model Service returns to `HEALTHY` state.

### Rollback Considerations

- The system should store (or tag) previous versions and their environment variables.
- Rollback can be triggered automatically upon critical failure or manually if errors are detected.
- Upon rollback, the system reverts `traffic_ratio` and resources to the last known good deployment.

### Canary Support (Future Extension)

- Introduce a subset of new routes (e.g., 5-10% traffic) to test the new version in production-like conditions.
- If error rates exceed a defined threshold, the canary is rolled back before full deployment.
- This feature would require additional monitoring and alerting capabilities to be fully effective.

## Conclusion

Introducing rolling updates and blue-green deployments will help achieve near-zero downtime, minimize risk during releases, and allow for dynamic configuration changes. These enhancements will significantly improve the deployment experience for both customers and end users of the Backend.AI Model Service.

### References

- Kubernetes - Performing a Rolling Update: https://kubernetes.io/docs/tutorials/kubernetes-basics/update/update-intro/
> Rolling updates allow Deployments to take place with zero downtime by incrementally updating Pods instances with new ones.
