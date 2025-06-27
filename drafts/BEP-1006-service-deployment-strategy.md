---
Author: Jeongseok Kang (jskang@lablup.com)
Status: Draft
Created: 2025-06-19
---

# Backend.AI Model Service Deployment Strategy Enhancement

## Abstract

This proposal aims to enhance the model service deployment strategy by adopting widely used methods such as [Rolling Update](https://en.wikipedia.org/wiki/Rolling_release) and [Blue-Green Deployment](https://en.wikipedia.org/wiki/Blue%E2%80%93green_deployment). These improvements will enable safer updates with minimal downtime and more flexible configuration changes during deployments.

## AS-IS

- The current deployment process does not fully support advanced strategies like rolling updates or blue-green deployments.
- Environment variables (envs) are fixed at deployment time and cannot be modified dynamically during an update.

## TO-BE

1. Enable updating of envs during service deployment:
  - Allow environment variables to be modified dynamically as part of a rolling update or blue-green deployment, reducing the need for manual restarts.
2. Introduce Rolling Update strategy:
  - Implement step-wise updates to running services, ensuring that a portion of instances remains available while new instances are rolled out.
3. Support Blue-Green Deployment:
  - Provide a mechanism to deploy new versions alongside the current version and switch traffic seamlessly, enabling zero-downtime releases and easy rollbacks.
