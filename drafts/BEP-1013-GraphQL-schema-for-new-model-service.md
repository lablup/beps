---
Author: Bo Keum Kim (bkkim@lablup.com)
Status: Draft
Created: 2025-07-14
---

# Title

GraphQL Schema for new Model Service

## Motivation

Following up of [BEP-1009 Model Serving Registry](https://github.com/lablup/beps/blob/main/drafts/BEP-1009-model-serving-registry.md), this BEP proposes a comprehensive GraphQL schema for the new model serving service in Backend.AI

## Proposed Design

The GraphQL schema introduces two primary entities and their supporting types to manage model serving infrastructure:

### Core Entities

1. **ModelServingDeployment**: The top-level entity representing a model service deployment. It manages:
   - Multiple revisions of a model
   - Traffic routing between revisions
   - Public endpoint configuration
   - Cluster configuration and scaling groups
   - Domain and project associations
   - Traffic distribution between revisions

2. **ModelServingRevision**: Represents a specific version of a model within a deployment. It handles:
   - Container image and runtime configuration
   - Model storage and mounting
   - Resource allocation and scaling
   - Service-specific configurations

## Technical Details

### ModelServingDeployment

A deployment is the top-level concept that manages multiple revisions.

#### Basic Fields
- `id`: Unique identifier
- `name`: Deployment name (unique within domain)
- `endpointUrl`: Public URL for service access (e.g., https://llama-3.model.ai)
- `preferredDomainName`(optional): Custom domain name preference
- `status`: Deployment status
  - ACTIVE, HIBERNATED, PROVISIONING, UPDATING, FAILED, DESTROYING, DESTROYED
- `openToPublic`: Whether accessible to public
- `tags`: Tags for model service

#### Cluster Configuration
- `clusterMode`: Cluster mode ('single-node' or 'multi-node')
- `clusterSize`: Number of nodes in cluster

#### Relationship Fields
- `domain`: Backend.AI domain
- `project`: Owning project (group)
- `createdUser`: User who created the deployment
- `revisions`: All revisions belonging to this deployment
- `accessTokens`: Access Tokens for endpoint url

#### Metadata
- `createdAt`: Creation timestamp
- `updatedAt`: Last update timestamp

### ModelServingRevision

A revision represents a specific version of a model service.

#### Basic Fields
- `id`: Unique identifier
- `name`: Revision name
- `tag`: Tag for revision
- `status`: Revision status (e.g. PENDING, PROVISIONING, HEALTHY, etc...)

#### Resource Configuration
- `resourceSlots`: Resource requirements (e.g., {"cuda.device": 2, "mem": "48g", "cpu": 8})
- `resourceOpts`: Additional resource options (e.g., {"shmem": "64m"})

#### Redeployment Strategy
- `redeploymentStrategy`: Strategy for redeployment
  - `type`: ROLLING, BLUE_GREEN, CANARY, RECREATE (Can be implemented with Enum)
  - `config`: Strategy-specific configuration (JSON)

#### Runtime, Model, Service Configuration
- `image`: Container image to use
  - `name`: Name of container image
  - `architecture`: CPU architecture (x86_64, aarch64)
- `runtimeVariant`: Runtime type (vllm, tgi, custom)
- `modelFolderId`: VFolder ID containing model files
- `modelMountDestination`: Model mount path in container (default: /models)
- `modelDefinitionPath`: Model definition file path (default: model-definition.yaml)
- `servingConfig`: Service-specific configuration options in JSON format.  
  - For vLLM: `max_model_length`, `parallelism` (`pp_size`, `tp_size`), `extra_cli_parameters`
- `environ`: Environment variables to be set in the container, provided as a JSON object
- `extraMounts`: List of additional volume mount configurations
  - Each mount contains: `vfolderId`, `mountDestination`, `type`, `permission` (optional)

#### Traffic and Scaling
- `trafficRatio`: Traffic percentage allocated to this revision (0.0-1.0)
- `desiredReplicas`: Target replica count
- `replicas`: Current running replica count

#### Relationship Fields
- `deployment`: Parent deployment
- `routings`: Routing entries for this revision
- `autoScalingRules`: Auto-scaling rules
- `scalingGroup`: Scaling Group for model deployment

#### Metadata
- `errorData`: Error information if failed
- `createdAt`: Creation timestamp
- `updatedAt`: Last update timestamp


## Input Types

### CreateModelServingDeploymentInput
Fields required for creating a new deployment:
- `name`: Deployment name
- `preferredDomainName`(optional): Custom domain
- `domain`: Backend.AI domain
- `project`: Project (group) ID or name
- `openToPublic`: Whether publicly accessible
- `clusterMode`: Cluster mode
- `clusterSize`: Cluster size
- `redeploymentStrategy`: Deployment strategy configuration
- `initialRevision`: Initial revision configuration

### CreateModelServingRevisionInput
Fields required for creating a new revision:
- `name`(optional): Revision name
- `tag`(otional): Revision tag
- `image`: Container image information
  - `name`: Image name with tag
  - `architecture`: Architecture of CPU
- `runtimeVariant`: Runtime type
- `modelFoldereId`: Model VFolder ID
- `modelMountDestination`: Model mount path
- `modelDefinitionPath`: Model definition file path
- `resourceSlots`: Resource requirements
- `resourceOpts`(optional): Additional resource options
- `servingConfig`: Service configuration
- `environ`(optional): Environment variables
- `extraMounts`(optional): List of additional mount configurations
- `desiredReplicas`: Initial replica count


## Query Examples

### 1. Get Deployment Details

```graphql
query GetDeploymentDetails {
  modelServingDeployment(id: "deployment-uuid") {
    id
    name
    endpointUrl
    preferredDomainName
    status
    openToPublic
    tags
    clusterMode
    clusterSize
    domain {
      name
    }
    project {
      name
    }
    createdUser {
      email
      accessKey
    }
    redeploymentStrategy {
      type
      config
    }
    revisions(first: 10) {
      edges {
        node {
          id
          name
          tag
          status
          trafficRatio
          replicas
          desiredReplicas
          image {
            name
            architecture
          }
          runtimeVariant
          modelFolderId
          modelMountDestination
          modelDefinitionPath
          scalingGroup {
            name
          }
          servingConfig
          environ
          extraMounts {
            vfolderId
            mountDestination
            type
            permission
          }
          routings(first: 5) {
            edges {
              node {
                id
                status
                session {
                  id
                  status
                }
              }
            }
          }
        }
      }
      pageInfo {
        hasNextPage
        hasPreviousPage
      }
      totalCount
    }
    createdAt
    updatedAt
  }
}
```

### 2. List Deployments with Filters

```graphql
query ListDeployments {
  modelServingDeployments(
    first: 20
    filter: "status == 'ACTIVE' AND openToPublic == true"
    order: "created_at DESC"
  ) {
    edges {
      node {
        id
        name
        endpointUrl
        status
        tags
        openToPublic
        clusterMode
        revisions(first: 1, filter: "status == 'HEALTHY'") {
          edges {
            node {
              name
              tag
              replicas
              trafficRatio
            }
          }
        }
      }
    }
    totalCount
  }
}
```

### 3. Get Specific Revision Details

```graphql
query GetRevisionDetails {
  modelServingRevision(id: "revision-uuid") {
    id
    name
    tag
    status
    trafficRatio
    replicas
    desiredReplicas
    deployment {
      id
      name
      endpointUrl
    }
    image {
      name
      architecture
    }
    runtimeVariant
    modelFolderId
    resourceSlots
    resourceOpts
    servingConfig
    environ
    extraMounts {
      vfolderId
      mountDestination
      type
      permission
    }
    routings {
      edges {
        node {
          id
          status
          trafficRatio
        }
      }
    }
    autoScalingRules {
      edges {
        node {
          id
          metricType
          threshold
        }
      }
    }
    errorData
    createdAt
    updatedAt
  }
}
```

## Mutation Examples

### 1. Creating a Simple Deployment

```graphql
mutation CreateSimpleDeployment {
  createModelServingDeployment(input: {
    name: "llama-3-service"
    domain: "default"
    project: "ml-team-project-id"
    openToPublic: true
    tags: ["production", "llm"]
    clusterMode: "single-node"
    clusterSize: 1
    redeploymentStrategy: {
      type: RECREATE
      config: ""
    }
    initialRevision: {
      name: "initial"
      tag: "v1"
      image: {
        name: "vllm:0.9.1"
        architecture: "x86_64"
      }
      runtimeVariant: "vllm"
      modelFolderId: "eeb8c377-15d2-4a16-8ed8-01215f3a5353"
      modelMountDestination: "/models"
      modelDefinitionPath: "model-definition.yaml"
      scalingGroup: "gpu-cluster"
      resourceSlots: "{\"cuda.device\": 2, \"mem\": \"48g\", \"cpu\": 8}"
      resourceOpts: "{\"shmem\": \"64m\"}"
      servingConfig: "{\"max_model_length\": 4096, \"parallelism\": {\"pp_size\": 1, \"tp_size\": 2}}"
      environ: "{\"CUDA_VISIBLE_DEVICES\": \"0,1\"}"
      extraMounts: [
        {
          vfolderId: "550e8400-e29b-41d4-a716-446655440001"
          mountDestination: "/data"
          type: "bind"
        }
      ]
      desiredReplicas: 2
    }
  }) {
    success
    message
    deployment {
      id
      name
      endpointUrl
      status
      tags
      revisions(first: 1) {
        edges {
          node {
            id
            name
            tag
            status
            replicas
          }
        }
      }
    }
  }
}
```

### 2. Creating an Expert Mode Deployment with Canary Strategy

```graphql
mutation CreateExpertDeployment {
  createModelServingDeployment(input: {
    name: "falcon-7b-optimized"
    preferredDomainName: "falcon.mycompany.ai"
    domain: "default"
    project: "research-team-id"
    openToPublic: false
    tags: ["experimental", "falcon"]
    clusterMode: "multi-node"
    clusterSize: 3
    redeploymentStrategy: {
      type: CANARY
      config: "{\"canaryPercentage\": 10, \"canaryDuration\": \"30m\", \"successThreshold\": 95}"
    }
    initialRevision: {
      name: "baseline"
      tag: "v1.0"
      image: {
        name: "python-tcp-app:3.9-ubuntu20.04"
        architecture: "x86_64"
      }
      runtimeVariant: "custom"
      modelFolderId: "550e8400-e29b-41d4-a716-446655440000"
      modelMountDestination: "/models"
      modelDefinitionPath: "model-definition.yaml"
      scalingGroup: "gpu-premium"
      resourceSlots: "{\"cuda.device\": 4, \"mem\": \"96g\", \"cpu\": 16}"
      resourceOpts: "{\"shmem\": \"128m\"}"
      servingConfig: "{\"extra_cli_parameters\": \"--trust-remote-code --enable-lora --gpu-memory-utilization 0.95\"}"
      environ: "{\"CUDA_VISIBLE_DEVICES\": \"0,1,2,3\", \"HF_TOKEN\": \"hf_xxx\"}"
      extraMounts: [
        {
          vfolderId: "7a83e195-7410-4768-a338-a949cef6be83"
          mountDestination: "/home/work/datasets"
          type: "bind"
        }
      ]
      desiredReplicas: 4
    }
  }) {
    success
    deployment {
      id
      name
      endpointUrl
      preferredDomainName
      clusterMode
      clusterSize
      tags
      redeploymentStrategy {
        type
        config
      }
    }
  }
}
```

### 3. Creating a New Revision

```graphql
mutation CreateNewRevision {
  createModelServingRevision(
    deploymentId: "deployment-uuid"
    input: {
      name: "optimized-version"
      tag: "v2.0"
      image: {
        name: "vllm:0.9.2"
        architecture: "x86_64"
      }
      runtimeVariant: "vllm"
      modelFolderId: "550e8400-e29b-41d4-a716-446655440000"
      modelMountDestination: "/models"
      modelDefinitionPath: "model-definition.yaml"
      resourceSlots: "{\"cuda.device\": 4, \"mem\": \"96g\", \"cpu\": 16}"
      resourceOpts: "{\"shmem\": \"128m\"}"
      servingConfig: "{\"max_model_length\": 8192, \"parallelism\": {\"pp_size\": 2, \"tp_size\": 4}, \"extra_cli_parameters\": \"--enable-lora\"}"
      environ: "{\"CUDA_VISIBLE_DEVICES\": \"0,1,2,3\"}"
      extraMounts: []
      desiredReplicas: 4
    }
  ) {
    success
    revision {
      id
      name
      tag
      status
      deployment {
        id
        name
      }
    }
  }
}
```

### 4. Managing Traffic Distribution Between Revisions

```graphql
mutation UpdateTrafficRatio {
  updateRevisionTrafficRatio(
    deploymentId: "deployment-uuid"
    trafficDistribution: [
      { revisionId: "stable-revision-id", ratio: 0.8 }
      { revisionId: "canary-revision-id", ratio: 0.2 }
    ]
  ) {
    success
    affectedRevisions {
      id
      name
      tag
      trafficRatio
      replicas
      status
    }
  }
}
```

### 5. Scaling a Revision

```graphql
mutation ScaleRevision {
  scaleRevision(
    revisionId: "revision-uuid"
    replicas: 6
  ) {
    success
    revision {
      id
      name
      tag
      desiredReplicas
      replicas
      status
      resourceSlots
    }
  }
}
```

### 6. Update Redeployment Configuration

```graphql
mutation UpdateRedeploymentStrategy {
  updateModelServingDeployment(
    id: "deployment-uuid"
    input: {
      redeploymentStrategy: {
        type: ROLLING
        config: "{\"maxSurge\": 2, \"maxUnavailable\": 1}"
      }
    }
  ) {
    success
    deployment {
      id
      name
      redeploymentStrategy {
        type
        config
      }
    }
  }
}
```

## Impacts to Users or Developers

- **For Users:**  
    Users will benefit from a more flexible and robust model serving experience, including easier deployment management, versioning, and scaling. The public endpoint and domain configuration features simplify access and integration.

- **For Developers:**  
    Developers gain a well-structured GraphQL API for managing model deployments and revisions. The schema supports advanced deployment strategies (e.g., canary, blue-green), resource configuration, and traffic management, reducing operational complexity.

## References

- [BEP-1009 Model Serving Registry](https://github.com/lablup/beps/blob/main/drafts/BEP-1009-model-serving-registry.md)
- [GraphQL Specification](https://spec.graphql.org/)
