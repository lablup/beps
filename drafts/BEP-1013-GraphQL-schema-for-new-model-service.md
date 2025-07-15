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

1. **ModelDeployment**: The top-level entity representing a model service deployment. It manages:
   - Active revision and revision history
   - Deployment strategy (Rolling, Blue-Green, Canary)
   - Public endpoint configuration and custom domain
   - Cluster configuration and resource groups
   - Domain and project associations
   - Replica management with auto-scaling rules

2. **ModelRevision**: Represents an immutable version of a model within a deployment. It handles:
   - Container image and runtime configuration
   - Model storage and mounting configuration
   - Resource requirements and runtime variant
   - Service-specific configurations
   - Additional volume mounts

## Technical Details

### ModelDeployment

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

#### Replica Configuration
- `replica`:
    - `desiredReplicaCount`: Number of replicas to be achieved
    - `replicas`: List of `ModelReplica`s managed by deployment 

#### Deployment Strategy
- `deploymentStrategy`
    - `type`: Types of Model Deploy Strategy
    - `config`: Configs needed for deployment strategy (e.g., maxSurge)

#### Revision Information
- `revision`: Current active ModelRevision that deployment references
- `revisionHistory`: List of previous ModelRevisions deployment referenced

#### Cluster Configuration
- `clusterConfig`:
    - `clusterMode`: Cluster mode ('single-node' or 'multi-node')
    - `clusterSize`: Number of nodes in cluster

#### Relationship Fields
- `domain`: Backend.AI domain
- `project`: Owning project (group)
- `createdUser`: User who created the deployment
- `revisions`: All revisions belonging to this deployment
- `accessTokens`: Access Tokens for endpoint url
- `resourceGroup`: Resource Group for deployment

#### Metadata
- `createdAt`: Creation timestamp
- `updatedAt`: Last update timestamp

#### Schema
```graphql
enum DeploymentStatus {
    INACTIVE
    ACTIVE
}

enum ClusterMode {
    SINGLE_NODE
    MULTI_NODE
}

type ClusterConfig {
    clusterMode: ClusterMode!
    clusterSize: Int!
}

type ReplicaManagement {
    desiredReplicaCount: Int!
    replicas: [ModelReplica!]!
    autoScalingRules: [AutoScalingRule!]!
}

type ModelReplica {
    name: String!
    revision: ModelRevision!
}

enum DeploymentStrategyType {
    ROLLING
    BLUE_GREEN
    CANARY
}

union StrategyConfig = RollingConfig | BlueGreenConfig | CanaryConfig

type DeploymentStrategy {
    type: DeploymentStrategyType!
    config: StrategyConfig
}

type ModelDeployment {
    id: ID!
    name: String!
    endpointUrl: String
    preferredDomainName: String
    status: DeploymentStatus!
    openToPublic: Boolean!
    tags: [String!]!

    revision: ModelRevision
    revisionHistory: [ModelRevision!]!
    
    replicaConfig: ReplicaManagement!
    clusterConfig: ClusterConfig!

    deploymentStrategy: DeploymentStrategy!

    domain: Domain!
    project: Project!
    createdUser: User!
    resourceGroup: ResourceGroup!
    revisions: [ModelRevision!]!
    accessTokens: [AccessToken!]!

    createdAt: DateTime!
    updatedAt: DateTime!
}
```

#### Query & Subscription
```graphql

type ReplicaMetric {
  ... # TODO: Fill up metric fields
}

type DeploymentMetrics {
    replicaMetrics: [ReplicaMetric!]!
}

type Query {
    # Deployment Queries
    deployments(filter: DeploymentFilter, limit: Int, offset: Int): [ModelDeployment!]!
    deployment(id: ID!): Deployment
 
    # Replica Queries
    replica(id: ID!): ModelReplica

    # Metrics
    deploymentMetrics(id: ID!, filter: DeploymentMetricsFilter): [DeploymentMetrics!]!
}

type DeploymentMetrics {
    replicaMetrics: [ReplicaMetric!]!
}

type Subscription {
    # Real-time Updates
    deploymentStatusChanged(deploymentId: ID!): ModelDeployment!
    replicaStatusChanged(revisionId: ID!): ModelReplica!
    metricsUpdated(deploymentId: ID!): DeploymentMetrics!
}

```

#### Mutation
```graphql

type Mutation {
    # Deployment Management
    createModelDeployment(input: CreateModelServingDeploymentInput! ): CreateModelDeploymentPayload!
    updateModelDeployment(input: UpdateModelServingDeploymentInput! ): UpdateModelDeploymentPayload!
    deleteModelServingDeployment(id: ID!): ID!
}

```

### ModelRevision

A revision represents a specific version of a model service. Revision is immutable component. If you want to update certain value of revision, user must publish new revision

#### Basic Fields
- `id`: Unique identifier
- `name`: Revision name (optional)
- `tags`: List of revision tags
- `status`: Revision status (ACTIVE, INACTIVE)

#### Resource and Runtime Configuration
- `resourceConfig`: Resource requirements and additional options
    - `resourceSlots`: Required resource slot information (e.g., {"cuda.device": 2, "mem": "48g", "cpu": 8})
    - `resourceOpts`: Additional resource options (e.g., {"shmem": "64m"})
- `ModelRuntimeConfig`: Runtime type and service configuration
    - `runtimeVariant`: Runtime type (VLLM, SGLANG, NVIDIA, MOJO, etc.)
    - `serviceConfig`: Service-specific configuration (e.g., for vLLM: max_model_length, parallelism, etc.)
    - `environ`: Container environment variables (JSON)

#### Model and Mount Configuration
- `modelVFolderConfig`: Model file and mount information
    - `vfolder`: Virtual Folder where the model is stored
    - `mountDestination`: Mount path inside the container (default: /models)
    - `definitionPath`: Model definition file path (default: model-definition.yaml)
    - `mounts`: List of additional volume mounts (each item: vfolderId, destination, type, permission)

#### Relationships
- `image`: Container image information used
- `routings`: Routing information for this revision

#### Metadata
- `errorData`: Error information if failed
- `createdAt`: Creation timestamp

#### Schema
```graphql
scalar JSONString

enum RuntimeVariant {
    VLLM,
    SGLANG,
    NVIDIA
    MOJO
}

enum MountPermission {
    READ_ONLY
    READ_WRITE
    RW_DELETE
}

enum RevisionStatus {
    ACTIVE
    INACTIVE
}

type Mount {
    vfolderId: ID!
    destination: String!
    type: MountType!
    permission: MountPermission!
}

type ModelVFolderConfig {
    vfolder: VirtualFolderNode!
    mountDestination: String!
    definitionPath: String!
    mounts: [Mount!]!
}

type ResourceConfig {
    resourceSlots: JSONString!
    resourceOpts: JSONString
}

union ServiceConfig = VLLMServiceConfig | SGLANGServiceConfig | NVIDIAServiceConfig | MOJOServiceConfig | CustomServiceConfig

type ModelRuntimeConfig {
    runtimeVariant: RuntimeVariant!
    serviceConfig: ServiceConfig
    environ: JSONString
}

type ModelRevision {
    id: ID!
    name: String
    tags: [String!]!
    status: RevisionStatus!

    resourceConfig: ResourceConfig!
    modelRuntimeConfig: ModelRuntimeConfig!
    modelVFolderConfig: ModelVFolderConfig!
 
    # Relationships
    image: Image!
    routings(first: Int, after: String): RoutingConnection!
    
    # Error and Metadata
    errorData: JSONString
    createdAt: DateTime!
}

```

#### Query & Subscription

```graphql
type Query {
    # Revision Queries
    revision(id: ID!): ModelRevision
    revisions(filter: ModelRevisionFilter, order: ModelRevisionOrder, first, after): [Revision!]!
}

type Subscription {
  # Real-time Updates
  deploymentStatusChanged(deploymentId: ID!): Deployment!
  replicaStatusChanged(revisionId: ID!): ReplicaInstance!
  metricsUpdated(deploymentId: ID!): DeploymentMetrics!
}

```

#### Mutation
```graphql
type Mutation {
    createModelRevision(input: CreateModelRevisionInput! ): CreateModelRevisionPayload!
    deleteModelRevision(id: ID!): ID!
}
```


## Input Types

### CreateModelDeploymentInput
Fields required for creating a new deployment:
- `name`: Deployment name (unique within domain)
- `preferredDomainName`(optional): Custom domain preference
- `domain`: Backend.AI domain name or ID
- `project`: Project (group) ID or name
- `openToPublic`: Whether publicly accessible
- `tags`(optional): Tags for model service
- `clusterConfig`: Cluster configuration
  - `clusterMode`: Single-node or multi-node
  - `clusterSize`: Number of nodes in cluster
- `deploymentStrategy`: Deployment strategy configuration
  - `type`: ROLLING, BLUE_GREEN, or CANARY
  - `config`: Strategy-specific configuration
- `initialRevision`: Initial revision configuration
- `resourceGroup`: Resource group for deployment

### CreateModelRevisionInput
Fields required for creating a new revision:
- `name`(optional): Revision name
- `tags`(optional): List of revision tags
- `image`: Container image information
  - `name`: Image name with tag
  - `architecture`: CPU architecture
- `modelRuntimeConfig`: Runtime configuration
  - `runtimeVariant`: VLLM, SGLANG, NVIDIA, MOJO, or custom
  - `serviceConfig`: Service-specific configuration
  - `environ`(optional): Environment variables (JSON)
- `modelVFolderConfig`: Model folder configuration
  - `vfolderId`: Model VFolder ID
  - `mountDestination`: Model mount path (default: /models)
  - `definitionPath`: Model definition file path
  - `mounts`(optional): Additional volume mounts
- `resourceConfig`: Resource configuration
  - `resourceSlots`: Resource requirements (JSON)
  - `resourceOpts`(optional): Additional resource options (JSON)


### 1. Get Deployment Details

```graphql
query GetDeploymentDetails {
  deployment(id: "deployment-uuid") {
    id
    name
    endpointUrl
    preferredDomainName
    status
    openToPublic
    tags
    
    revision {
      id
      name
      tags
      status
    }
    
    revisionHistory {
      id
      name
      tags
      createdAt
    }
    
    replicaConfig {
      desiredReplicaCount
      replicas {
        name
        revision {
          id
          name
        }
      }
      autoScalingRules {
        id
        metricType
        threshold
      }
    }
    
    clusterConfig {
      clusterMode
      clusterSize
    }
    
    deploymentStrategy {
      type
      config
    }
    
    domain {
      name
    }
    
    project {
      name
    }
    
    createdUser {
      email
    }
    
    resourceGroup {
      name
    }
    
    createdAt
    updatedAt
  }
}
```

### 2. List Deployments with Filters

```graphql
query ListDeployments {
  deployments(
    filter: {
      status: ACTIVE
      openToPublic: true
    }
    limit: 20
    offset: 0
  ) {
    id
    name
    endpointUrl
    status
    tags
    openToPublic
    
    revision {
      id
      name
      tags
      status
    }
    
    clusterConfig {
      clusterMode
      clusterSize
    }
    
    replicaConfig {
      desiredReplicaCount
      replicas {
        name
      }
    }
  }
}
```

### 3. Get Specific Revision Details

```graphql
query GetRevisionDetails {
  revision(id: "revision-uuid") {
    id
    name
    tags
    status
    
    resourceConfig {
      resourceSlots
      resourceOpts
    }
    
    modelRuntimeConfig {
      runtimeVariant
      serviceConfig
      environ
    }
    
    modelVFolderConfig {
      vfolder {
        id
        name
      }
      mountDestination
      definitionPath
      mounts {
        vfolderId
        destination
        type
        permission
      }
    }
    
    image {
      name
      architecture
    }
    
    routings(first: 10) {
      edges {
        node {
          id
          status
        }
      }
    }
    
    errorData
    createdAt
  }
}
```

## Mutation Examples

### 1. Creating a Simple Deployment

```graphql
mutation CreateSimpleDeployment {
  createModelDeployment(input: {
    name: "llama-3-service"
    domain: "default"
    project: "ml-team-project-id"
    openToPublic: true
    tags: ["production", "llm"]
    clusterConfig: {
      clusterMode: SINGLE_NODE
      clusterSize: 1
    }
    deploymentStrategy: {
      type: ROLLING
      config: {
        maxSurge: 1
        maxUnavailable: 0
      }
    }
    resourceGroup: "gpu-cluster"
    initialRevision: {
      name: "initial"
      tags: ["v1.0", "stable"]
      image: {
        name: "vllm:0.9.1"
        architecture: "x86_64"
      }
      modelRuntimeConfig: {
        runtimeVariant: VLLM
        serviceConfig: {
          maxModelLength: 4096
          parallelism: {
            ppSize: 1
            tpSize: 2
          }
        }
        environ: "{\"CUDA_VISIBLE_DEVICES\": \"0,1\"}"
      }
      modelVFolderConfig: {
        vfolderId: "eeb8c377-15d2-4a16-8ed8-01215f3a5353"
        mountDestination: "/models"
        definitionPath: "model-definition.yaml"
        mounts: [
          {
            vfolderId: "550e8400-e29b-41d4-a716-446655440001"
            destination: "/data"
            type: BIND
            permission: READ_ONLY
          }
        ]
      }
      resourceConfig: {
        resourceSlots: "{\"cuda.device\": 2, \"mem\": \"48g\", \"cpu\": 8}"
        resourceOpts: "{\"shmem\": \"64m\"}"
      }
    }
  }) {
    deployment {
      id
      name
      endpointUrl
      status
      tags
      revision {
        id
        name
        tags
        status
      }
    }
  }
}
```

### 2. Creating an Expert Mode Deployment with Canary Strategy

```graphql
mutation CreateExpertDeployment {
  createModelDeployment(input: {
    name: "falcon-7b-optimized"
    preferredDomainName: "falcon.mycompany.ai"
    domain: "default"
    project: "research-team-id"
    openToPublic: false
    tags: ["experimental", "falcon"]
    clusterConfig: {
      clusterMode: MULTI_NODE
      clusterSize: 3
    }
    deploymentStrategy: {
      type: CANARY
      config: {
        canaryPercentage: 10
        canaryDuration: "30m"
        successThreshold: 95
      }
    }
    resourceGroup: "gpu-premium"
    initialRevision: {
      name: "baseline"
      tags: ["v1.0", "baseline"]
      image: {
        name: "python-tcp-app:3.9-ubuntu20.04"
        architecture: "x86_64"
      }
      modelRuntimeConfig: {
        runtimeVariant: CUSTOM
        serviceConfig: {
          extraCliParameters: "--trust-remote-code --enable-lora --gpu-memory-utilization 0.95"
        }
        environ: "{\"CUDA_VISIBLE_DEVICES\": \"0,1,2,3\", \"HF_TOKEN\": \"hf_xxx\"}"
      }
      modelVFolderConfig: {
        vfolderId: "550e8400-e29b-41d4-a716-446655440000"
        mountDestination: "/models"
        definitionPath: "model-definition.yaml"
        mounts: [
          {
            vfolderId: "7a83e195-7410-4768-a338-a949cef6be83"
            destination: "/home/work/datasets"
            type: BIND
            permission: READ_WRITE
          }
        ]
      }
      resourceConfig: {
        resourceSlots: "{\"cuda.device\": 4, \"mem\": \"96g\", \"cpu\": 16}"
        resourceOpts: "{\"shmem\": \"128m\"}"
      }
    }
  }) {
    deployment {
      id
      name
      endpointUrl
      preferredDomainName
      tags
      clusterConfig {
        clusterMode
        clusterSize
      }
      deploymentStrategy {
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
  createModelRevision(input: {
    deploymentId: "deployment-uuid"
    name: "optimized-version"
    tags: ["v2.0", "optimized"]
    image: {
      name: "vllm:0.9.2"
      architecture: "x86_64"
    }
    modelRuntimeConfig: {
      runtimeVariant: VLLM
      serviceConfig: {
        maxModelLength: 8192
        parallelism: {
          ppSize: 2
          tpSize: 4
        }
        extraCliParameters: "--enable-lora"
      }
      environ: "{\"CUDA_VISIBLE_DEVICES\": \"0,1,2,3\"}"
    }
    modelVFolderConfig: {
      vfolderId: "550e8400-e29b-41d4-a716-446655440000"
      mountDestination: "/models"
      definitionPath: "model-definition.yaml"
      mounts: []
    }
    resourceConfig: {
      resourceSlots: "{\"cuda.device\": 4, \"mem\": \"96g\", \"cpu\": 16}"
      resourceOpts: "{\"shmem\": \"128m\"}"
    }
  }) {
    revision {
      id
      name
      tags
      status
      createdAt
    }
  }
}
```

### 4. Updating Deployment Configuration

```graphql
mutation UpdateDeployment {
  updateModelDeployment(input: {
    id: "deployment-uuid"
    openToPublic: true
    tags: ["production", "llm", "updated"]
    deploymentStrategy: {
      type: BLUE_GREEN
      config: {
        autoPromotionEnabled: true
        terminationWaitTime: 300
      }
    }
  }) {
    deployment {
      id
      name
      openToPublic
      tags
      deploymentStrategy {
        type
        config
      }
    }
  }
}
```

### 5. Switching Active Revision

```graphql
mutation SwitchActiveRevision {
  updateModelDeployment(input: {
    id: "deployment-uuid"
    activeRevisionId: "new-revision-uuid"
  }) {
    deployment {
      id
      name
      revision {
        id
        name
        tags
        status
      }
      revisionHistory {
        id
        name
        tags
      }
    }
  }
}
```

### 6. Deleting a Deployment

```graphql
mutation DeleteDeployment {
  deleteModelServingDeployment(id: "deployment-uuid") {
    deletedId
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
