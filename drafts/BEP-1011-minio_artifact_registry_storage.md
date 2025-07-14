---
Author: Gyubong Lee (gbl@lablup.com)
Status: Draft
Created: 2025-07-10
---

# Support MinIO as Artifact Registry Storage Backend

Let's add support for `MinIO` in the storage proxy as a storage backend for the artifact registry.

## Motivation

Let's integrate external storage into the storage proxy so that models, images, packages, and more can be stored and accessed by the Backend.AI scanner. We will use MinIO as the storage backend.

# Use Case

1. Client sends a request to the manager to rescan the artifact registry via a GQL mutation.

2. Manager rescans an external artifact registry (such as HuggingFace or an external manager), and the metadata of the models is stored in the database.

3. Call storage-proxy API for downloading the rescanned models to the storage.

4. The models and metadata files are downloaded to the subdirectory path corresponding to the revision ID of the bucket.

```mermaid
sequenceDiagram
    participant Client
    participant Manager
    participant ExternalRegistry
    participant Database
    participant StorageProxy
    participant MinIO
    
    Client->>+Manager: Request to rescan artifact registry (GQL mutation)
    Manager->>+ExternalRegistry: Rescan external artifact registry (e.g., HuggingFace)
    ExternalRegistry-->>-Manager: Return artifact metadata
    Manager->>+Database: Store model metadata and create revision ID
    Database-->>-Manager: Revision ID created
    Manager->>+StorageProxy: Call storage-proxy API for downloading models
    StorageProxy->>+MinIO: Download models to bucket subdirectory (revision ID path)
    MinIO-->>-StorageProxy: Models downloaded
    StorageProxy-->>-Manager: Download complete
    Manager-->>-Client: Rescan complete
```

# Database Schema

## Artifact Registry Storage Mapping Table

The artifact registry requires database tables to store mapping information between artifacts and MinIO bucket paths.

```sql
CREATE TABLE artifacts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    type VARCHAR(20) NOT NULL,  -- 'image', 'package', 'model'
    name VARCHAR(50) NOT NULL,
    storage_backend VARCHAR(50) NOT NULL DEFAULT 'minio',  -- 'minio', 's3', etc.
);

-- MinIO storage specific data
CREATE TABLE artifact_meta_minio (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(50) NOT NULL,
    bucket_name VARCHAR(63) NOT NULL,
    bucket_path TEXT NOT NULL,
);

-- Create separate tables for storing metadata for each storage type.
CREATE TABLE artifact_meta_s3 (
...
);

```

# Storage Proxy Implementations

The following additional implementations are required to realize the use cases.

## Integration with MinIO

Add service code to the storage proxy for configuring the artifact registry.
We can refer to components such as `BaseVolume` and `FsOpModel`, which operate as the VFolder backend, when writing the implementation. But, since the artifact registry storage does not need to function as a VFolder at this point, let's remove any unnecessary parts and properly abstract the interface.

Communication with `MinIO` is handled using `s3fs`. We will need to implement an `FsOpModel` that mounts the `s3fs` bucket as a file system on the storage host and handles download and upload operations.

## API Specifications

It will be necessary to implement additional CRUD REST APIs for each artifact type in the storage proxy.

### Storage Proxy REST APIs

The storage proxy APIs will provide general storage operations for managing content from external sources:

#### Rescan Metadata from External Registry

Rescans the metadata from an external registry. After the rescanned metadata is sent to the *Manager*, it will be stored in the database.

```
POST /storages/{storage_type}/rescan
Content-Type: application/json

{...}

Response:
{...}
```

#### Download Artifact Files from External Registry

Rescans the external registry and downloads the artifact files to the designated storage.

```
POST /storages/{storage_type}/download
Content-Type: application/json

{...}

Response:
{
  "task_id": "download_task_123",
  "status": "started",
  "message": "Download started to models/gpt-2/abc123def456"
}
```


#### Get Download Task Status

Returns the download task's status.

```
GET /storages/{storage_type}/download/{task_id}

Response:
{
  "task_id": "download_task_123",
  "status": "completed",
  "progress": 100,
  "target_path": "models/gpt-2/abc123def456",
  "downloaded_files": [
    "config.json",
    "pytorch_model.bin",
    "tokenizer.json"
  ]
}
```

#### List Storage Content

Lists the files located in a specific path of the designated storage and returns them in the response.

```
GET /storages/{storage_type}/list-files

Response:
{
  "path": "models/gpt-2/abc123def456",
  "size": 548000000,
  "files": [
    {
      "name": "config.json",
      "size": 1024,
      "last_modified": "2025-07-11T10:00:00Z"
    },
    {
      "name": "pytorch_model.bin",
      "size": 547998976,
      "last_modified": "2025-07-11T10:05:00Z"
    }
  ]
}
```

#### Delete Storage Content

Deletes the files located in a specific path of the designated storage.

```
DELETE /storages/{storage_type}

Response: 204 No Content
```

## Config format

```toml
[storages]

[storages.minio1]
backend = "minio"
endpoint = "https://minio.example.com"

[storages.minio1.options]
minio-access-key = "your-access-key"
minio-secret-key = "your-secret-key"

[storages.minio2]
backend = "minio"
endpoint = "https://minio.example2.com"

[storages.minio2.options]
minio-access-key = "your-access-key2"
minio-secret-key = "your-secret-key2"
```

# Reference

- [`s3fs` FAQ](https://github.com/s3fs-fuse/s3fs-fuse/wiki/FAQ)

- [`s3fs` options](https://github.com/s3fs-fuse/s3fs-fuse/wiki/Fuse-Over-Amazon#options)
