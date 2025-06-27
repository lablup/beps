---
Author: Jeongseok Kang (jskang@lablup.com)
Status: Draft
Created: 2025-06-27
---

# Backend.AI Model Service Deployment Strategy Enhancement

## Abstract

This proposal aims to enhance the model service deployment strategy by adopting [Rolling update](https://en.wikipedia.org/wiki/Rolling_release), one of the widely used methods. This improvement will enable safer updates with minimal downtime and more flexible configuration changes during deployments.

## Motivation
As the necessity for the model serving has been growing over time, Backend.AI has [introduced the 'Model Service'](https://www.backend.ai/ko/blog/2023-09-26-Backend.AI-23.09-update) in version 23.09 and Backend.AI FastTrack has [introduced the 'Model Serving Task'](https://github.com/lablup/backend.ai-fasttrack/releases/tag/25.9.0) in 25.09 to qualify the market's needs.

However, it has been found that the current deployment process does not fully support zero-downtime deployments, which is critical when integrating the Backend.AI Model Service into end user-facing services.

Therefore, Backend.AI aims to introduce rolling update, along with rollback of course.

## Design

1. User triggers or submits a promotion.
  1-1. If there is not enough resources to create new model service sessions for now, discard the submission.
2. sdfsd

### AS-IS

- The current deployment process does not fully support advanced strategies like rolling updates or blue-green deployments.
- Environment variables (envs) are fixed at deployment time and cannot be modified dynamically during an update.

### TO-BE

1. Introduce Rolling Update strategy:
  - Implement step-wise updates to running services, ensuring that a portion of instances remains available while new instances are rolled out.
2. Enable updating of envs during service deployment:
  - Allow environment variables to be modified dynamically as part of a rolling update or blue-green deployment, reducing the need for manual restarts.
3. Each `Routing` will have a new field: `version`.

## Conclusion

The advanced deployment process with rolling updates will enable the zero-downtime deployments and will satisfy both our customers and the end users with better UX.

### References
- [Kubernetes - Performaing a rolling update](https://kubernetes.io/docs/tutorials/kubernetes-basics/update/update-intro/)

> Rolling updates allow Deployments' update to take place with zero downtime by incrementally updating Pods instances with new ones.
> In Kubernetes, updates are versioned and any Deployment update can be reverted to a previous (stable) version.

```
Rolling updates allow the following actions:
- Promote an application from one environment to another (via container image updates)
- Rollback to previous versions
- Continuous Integration and Continuous Delivery of applications with zero downtime
```

---

[Blue-Green 배포 지원 방식]
1. API 호출을 통해 모델 서비스 업데이트 요청을 실행합니다. (UI 및 CLI 지원 예정)
  1-1. 현재 사용 가능한 자원이 충분하지 않을 경우 ('복제본 수'×'요청 자원량'보다 적을 경우) 요청을 처리하지 않고 에러 메시지를 반환합니다.
2. 모델 서비스는 `UPDATING` (가칭) 상태로 진입합니다.
3. 내부적으로 '복제본 수'(replicas) 만큼의 새 Green `Routing`을 생성합니다. 이때 새로 생성되는 Green Routing은 `traffic_ratio=0.0`으로 설정하여 요청을 전달받지 않도록 합니다.
4. '복제본 수' 만큼의 새 Green Routing이 모두 `HEALTHY` 상태가 되면 `traffic_ratio=1.0`로 업데이트하고 기존에 존재하던 Blue `Routing`들은 `traffic_ratio=0.0`로 업데이트합니다.
5. Blue `Routing`을 삭제하고 할당되었던 자원을 해제합니다.
6. 모델 서비스 `HEALTHY` 상태로 돌아옵니다.

* `traffic_ratio`는 각 `Routing`에 요청이 분배되는 비율을 의미합니다.

- 4번 단계에서 단순히 healthy 만으로 업데이트하는게 아니라 [model serving 에서 미리 조건을 받고, 모든 조건이 만족되었을 때 배포] 로 가는게 어떨까요? ex: 모두 healthy 시점, 배포 버튼을 누르면 배포 등...
- blue-green 배포 이후에 단순히 이전 routing 들 삭제해버리면 곤란할 수 있어서 롤백 하기 위한 구조도 고려 필요해보입니다.
- 혹은 카나리 배포도 지원하거나 ㅎㅎ (healthy 자체의 신뢰만으론 어려우니 요청을 받았을 때 문제 없는지 등등...)
