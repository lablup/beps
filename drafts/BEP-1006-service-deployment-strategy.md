---
Author: Jeongseok Kang (jskang@lablup.com)
Status: Draft
Created: 2025-06-27
---

# Backend.AI Model Service Deployment Strategy Enhancement

## Abstract

This proposal aims to enhance the Backend.AI Model Service deployment process with a rolling update strategy. Rolling updates allow incremental deployment of a new version while some instances of the service remain running, resulting in minimal downtime. The improved strategy also supports dynamic environment variable updates, rollback mechanisms, and versioning for better release management and reliability.

## Motivation
As model-serving usage has grown, Backend.AI introduced the “Model Service” in version 23.09 and Backend.AI FastTrack introduced the “Model Serving Task” in 25.09 to meet the demand for operational AI services. However, the existing deployment approach does not fully support zero-downtime or seamless updates, which are critical for production environments. To address this, we propose a rolling update mechanism that incrementally replaces older instances with new ones while keeping the service active.

## Design

### AS-IS

1. Current deployment methods are relatively basic, causing noticeable interruptions during upgrades.
2. Environment variables (envs) are fixed at deployment time and cannot be dynamically updated without a full redeployment.
3. Lack of versioned routing objects (e.g., no dedicated “version” field) makes it more challenging to roll back to a stable release if issues arise.

### TO-BE

1. Rolling Update Strategy:
- Users trigger an update or “promotion” when a new version needs to be deployed.
- If there are insufficient resources (e.g., if “replicas × resource-required” exceeds availability), the request is rejected to avoid partial or unstable deployments.
- The system incrementally spins up new instances. As each new instance becomes healthy, some traffic is directed to it.
- The process continues until all old instances have been replaced or until a failure triggers a rollback to the last stable version.

2. Dynamic Environment Variables:
- Allow changes to environment variables during a rolling update so that a full redeployment is not needed for routine configuration updates.
- For best results, the system gracefully reloads or restarts only the affected instances, trickling traffic to new pods as they become ready.

3. Enhanced Versioning:
- Introduce a version field to Routing to simplify release management and rollbacks.
- Store metadata about previous versions, enabling one-click or automated rollbacks.

### Detailed Flow for Rolling Updates

1. User triggers a promotion or update request for the Model Service.
2. The system begins creating new model service instances (e.g., pods or containers).
3. If resources are insufficient, the update is rejected with an error message.
4. Once each new instance is healthy, a fraction of traffic is allocated to it.
5. The system progressively scales traffic from older instances to newer ones.
6. If an error is detected (e.g., failing health checks, excessive errors in logs), the rollout is paused or rolled back.
7. On successful completion, all traffic is routed to the new version, and the old instances are retired.

### Rollback Considerations

- The system retains references to the previous stable version and associated environment variables.
- An automated or manual rollback can be initiated at any stage if a new version is found to be defective.
- Rollbacks reuse the stored version metadata, ensuring minimal downtime during revert operations.

### Canary (Future Possibility)

- As a future extension, canary releases could test a new version with a small subset of traffic before promoting it to full production.
- This approach would require additional monitoring and alerting to detect anomalies promptly.

## Conclusion

Introducing rolling updates significantly improves the robustness and reliability of Backend.AI Model Service deployments. The enhancements described—incremental instance replacement, dynamic environment variables, and version-based rollbacks—will result in near-zero downtime during new releases and simpler, safer deployment management.

### References

- Kubernetes - Performing a Rolling Update: https://kubernetes.io/docs/tutorials/kubernetes-basics/update/update-intro/
> Rolling updates allow Deployments to take place with zero downtime by incrementally updating Pods instances with new ones.
