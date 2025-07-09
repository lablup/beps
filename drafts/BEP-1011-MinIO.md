---
Author: Gyubong Lee (gbl@lablup.com)
Status: Draft
Created: 2025-07-09
---

# Integrate MinIO as Object Storage Model VFolder

## AS-IS

Backend.AI users can mount a bucket containing their desired model to their compute session using their S3 client, if the image supports it.

However, this is only possible if the image has the S3 client installed or if the user has the necessary permissions to install packages, which is inconvenient from the client's perspective.

## TO-BE

Let's add MinIO as a storage backend type for the storage proxy.

And let's use MinIO as the storage backend for VFolders created through that host.

This way, users will be able to conveniently reuse their model buckets regardless of the image or their permissions.

Users should be able to use their existing bucket data in the form of a vfolder on Backend.AI without any hassle.

# Implementations

To achieve this, the following tasks need to be carried out.

## Storage Proxy

Add MinIO storage backend type to support configurations like the one below.

```
[volume.minio]
backend = "minio"
endpoint = "https://minio.example.com"

[volume.minio.options]
minio-access-key = "your-access-key"
minio-secret-key = "your-secret-key"
```

> Although the combination of access key and secret key was used as an example, in reality, there may be requirements for a wider range of authentication methods such as IAM roles.

## Manager

When creating a session that mounts a MinIO vfolder, the manager must send the agent the credentials required to authenticate with the corresponding MinIO endpoint.

To do this, the manager will need to store the bucket (or bucket subdirectory) credentials corresponding to the vfolder somewhere in the database.

A simple approach could be to consider adding a column to the `vfolder` table to store metadata about the bucket mapping.

Each bucket's metadata must store, in addition to the credentials, values such as the endpoint, whether it is mounted as read-only, and the bucket path mapped to a subdirectory (only applicable when mapping a bucket subdirectory to a vfolder).

## Agent

Before creating a container, the agent lets `s3fs` mount the bucket into the local filesystem and bind-mount it into the container.

We need to keep track of the references to a specific bucket to prevent duplicate mounts and premature unmounts when there are multiple containers using the same bucket.

If the reference count to the bucket is zero after a container is destroyed, the agent could unmount the bucket.

# Considerations

The implementation itself may be relatively simple, but we need to consider various issues such as the following.

## Quota scope support

S3 operates on a `pay-as-you-go` model and does not have a built-in quota setting.

To implement QuotaScope on our side, we need to measure the size of a bucket or a subdirectory within the bucket.

However, this can also be challenging when there are a large number of files.

The default recursive command slows down significantly with many files, so you may need to consider alternative approaches.

Let's investigate what approaches vendors are taking with their storage solutions.

## Stability considerations

Various considerations regarding stability are necessary.

Wouldn't there be a potential scaling issue if hundreds of s3fs mounts are attached to the agent host?: From the perspective of mount stability, one possible approach could be to set a responsibility boundary by issuing a warning when the number exceeds a certain limit.

Various experiments and tests will be necessary to identify the actual issues that may arise in practice.

In addition, it is necessary to investigate whether there could be latency in cases where a session with the mounted bucket is being unmounted and another session including the same bucket is created simultaneously, and whether this can be controlled through detailed s3fs options.

Let's consider how tolerantly we need to design for these kinds of scenarios.

## Mapping VFolder to subdirectories of a bucket

It is also possible to map a subdirectory of a bucket, rather than the bucket itself, as a vfolder.

In this case, depending on the storage solution and vendor, there may be rate limits or concurrency restrictions, so it is necessary to verify this.

## RBAC and `s3fs` mounts managing

To control credentials by user, project, or individual folder level, it will likely be necessary to first develop RBAC (Role-Based Access Control) functionality.

Since RBAC itself is a separate issue, it will not be addressed here.

However, instead of mounting all the required `s3fs` on the agent host at once, it would be more efficient to retain only the buckets corresponding to the role of the container owner.

## VFolder invitation

In s3fs mounting, actual permissions will be aligned through the credentials.

However, since mount permissions can be arbitrarily specified using uid, gid, and umask, features like vfolder invitation or sharing can be implemented based on this.

Instead of directly integrating this implementation into our service, another alternative could be to guide users to manage permission grants at the API credential level within the MinIO (S3) service.

# Reference

- [VFolder abstraction for object storage (S3)](https://github.com/lablup/backend.ai/issues/665)

- [s3fs FAQ](https://github.com/s3fs-fuse/s3fs-fuse/wiki/FAQ)

- [s3fs options](https://github.com/s3fs-fuse/s3fs-fuse/wiki/Fuse-Over-Amazon#options)

