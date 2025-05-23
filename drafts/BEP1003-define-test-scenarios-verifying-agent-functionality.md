---
Author: Bokeum Kim (bkkim@lablup.com)
Status: Draft
---

# Agent Operations

---

## `update_scaling_group`
* **Description**: Updates the scaling group configuration for the agent.
* **Input Value**:
    * `scaling_group: str`: name of scaling-group
* **Response Value**:
    * `None` (implicitly on success).
* **Side Effects**:
    * Reads the agent's TOML configuration file.
    * Modifies the `agent.scaling-group` value in the configuration and Update file
    * Creates a backup of the original configuration file (e.g., `agent.toml.bak`).
    * Updates the `scaling-group` in the agent's in-memory `local_config`.

---

## `ping`
* **Description**: A simple ping RPC to check connectivity.
* **Input Value**: 
    * `msg: str`: ping message
* **Response Value**:
    * `str`: The original message sent in the request.

---

## `gather_hwinfo`
* **Description**: Collects hardware metadata from the agent's host.
* **Response Value**:
    * `Mapping[str, HardwareMetadata]`: A dictionary containing various hardware details (e.g., CPU, GPU, memory).

---

## `ping_kernel`
* **Description**: Pings a specific running kernel to check its responsiveness.
* **Input Value**:
    * `kernel_id: str`: Id of kernel that wants to send ping
* **Response Value**:
    * `dict[str, float] | None`: A dictionary with timing/status information if the kernel responds, otherwise `None`.

---

## `sync_kernel_registry`
* **Description**: Synchronizes the agent's kernel registry with a list provided by the manager. It terminates kernels on the agent that are not in the manager's list and flags kernels in the manager's list that are not on the agent.
* **Input Value**:
    * `raw_kernel_session_ids: Iterable[tuple[str, str]]`: list of tuples composed of kernel id and session id
* **Response Value**:
    * `None`.
* **Side Effects**:
    * For kernel IDs provided by the manager but not found in the agent's local registry:
        * Agent produces a `KernelTerminatedEvent` with reason `ALREADY_TERMINATED`.
    * For kernels in the agent's local registry but not in the list provided by the manager:
        * Calls `self.agent.inject_container_lifecycle_event` with `LifecycleEvent.DESTROY` and reason `NOT_FOUND_IN_MANAGER`.

---

## `check_and_pull`
* **Description**: Checks if specified container images exist locally and initiates a background task to pull them if they don't or if an update is needed.
* **Input Value**:
    * `image_configs: Mapping[str, ImageConfig]`: 
* **Response Value**:
    * `dict[str, str]`: A dictionary mapping image names to the background task IDs responsible for pulling them.
* **Side Effects**:
    * For each image:
        * A background task (`_pull`) is started.
        * The `_pull` task:
            * Checks local image presence and digest against `ImageConfig`.
            * If pulling is required:
                * Agent produces an `ImagePullStartedEvent`.
                * Calls `self.agent.pull_image()` to download the image from the specified registry.
                * Upon successful pull:
                    * Agent produces an `ImagePullFinishedEvent`.
                * Upon pull timeout or failure:
                    * Agent produces an `ImagePullFailedEvent`.
            * If pulling is not required (image already exists and is up-to-date):
                * Agent produces an `ImagePullFinishedEvent` (often with a message indicating it already exists).

---

## `create_kernels`
* **Description**: Creates one or more new computation kernels (containers).
* **Input Value**: 
    * `raw_session_id: str` 
    * `raw_kernel_ids: Sequence[str]`
    * `raw_configs: Sequence[dict]`
    * `raw_cluster_info: dict`
    * `kernel_image_refs: dict[KernelId, ImageRef]`
* **Response Value**:
    * `list[dict]`: A list of dictionaries, each containing information about a successfully created kernel (e.g., ID, host, ports, container ID, resource specs).
    * Raises an exception if any kernel creation fails (either the first error or a `TaskGroupError` for multiple failures).
* **Side Effects**:
    * For each set of kernel ID and kernel configuration provided:
        * Calls `self.agent.create_kernel()`, which involves the following flow and side effects:
            1.  **Preparation & Event**: produce a `KernelPreparingEvent` (if not restarting).
            2.  **Image Check & Pull**:
                * If pulling is necessary, emits a `KernelPullingEvent` and then 
                pulls the image from the registry
            3.  **Resource Allocation & Event**:
                * Produce a `KernelCreatingEvent` (if not restarting).
                * If resource allocation fails, agent produces `DoAgentResourceCheckEvent`
            4.  **Environment Setup**:
                * If not restarting, prepares scratch directories
                * Configures network for the kernel
                * Mounts virtual folders
            5.  **Configuration Persistence**: 
                * Store the cluster config to a kernel-related storage (e.g., scratch space)
            6.  **Container Creation**:
                * Prepare and start container
                * If container failed to prepare or start, `inject_container_lifecycle_event` is called with `LifecycleEvent.DESTROY` and reason `FAILED_TO_CREATE` and raise `AgentError`
            7.  **Kernel Initialization & Health Check**:
                * If the kernel fails to start or respond correctly within the timeout period, the `inject_container_lifecycle_event` is called with `LifecycleEvent.DESTROY` and reason `FAILED_TO_START` and raise `AgentError`
            8.  **Finalization & Event**:
                * If successful, updates the kernel's state to `RUNNING` in the `kernel_registry`.
                * Produces a `KernelStartedEvent` with details of the created kernel.
                * For inference sessions, may start a background task to monitor model service health.
    * Utilizes a semaphore (`kernel-creation-concurrency`) to limit concurrent kernel creations.

---

## `destroy_kernel`
* **Description**: Destroys a specific computation kernel (container).
* **Input Value**: 
    * `kernel_id: str`
    * `session_id: str`
    * `reason: Optional[KernelLifecycleEventReason]`
    * `suppress_events: bool`
* **Response Value**:
    * The result of an `asyncio.Future` (typically `None` on success, or an exception if destruction fails).
* **Side Effects**:
    * Calls `self.agent.inject_container_lifecycle_event` with `LifecycleEvent.DESTROY`

---

## `interrupt_kernel`
* **Description**: Sends an interrupt signal to a running kernel.
* **Input Value**: 
    * `kernel_id: str`
* **Response Value**:
    * `None`.
* **Side Effects**:
    * Calls `self.agent.interrupt_kernel()`, sending a interrup msg to kernel

---

## `get_completions`
* **Description**: Requests code completions from a kernel.
* **Input Value**: 
    * `kernel_id: str`
    * `text: str`
    * `opts: dict`
* **Response Value**:
    * JSON-parsed completion results containing suggested code completions

---

## `get_logs`
* **Description**: Retrieves logs from a specific kernel's container.
* **Input Value**: 
    * `kernel_id: str`
* **Response Value**:
    * JSON-parsed result of kernel logs

---

## `restart_kernel`
* **Description**: Restarts a kernel
* **Input Value**: 
    * `kernel_id: str`
    * `session_id: str`
    * `kernel_image: ImageRef`
    * `updated_config: dict`
* **Response Value**:
    * `dict[str, Any]`: Information about the newly restarted kernel (similar to `create_kernels` response for a single kernel).
* **Side Effects**:
    * Destroying old kerenl with given kernel id. call `inject_container_lifecycle_event` with `LifecycleEvent.DESTROY`and reason `RESTARTING`
        * If destroying event spent more than 60 seconds, `inject_container_lifecycle_event` called with `LifecycleEvent.CLEAN`and reason `RESTART_TIMEOUT` and Raise `asyncio.TimeoutError``
    * Restart kernel using `create_kernel`

---

## `execute`
* **Description**: Executes code within a specified kernel.
* **Input value**:
    * `session_id: str`
    * `kernel_id: str`
    * `api_version: int`
    * `run_id: str`
    * `mode: Literal["query", "batch", "continue", "input"]`
    * `code: str`
    * `opts: dict[str, Any]`
    * `flush_timeout: float`
* **Response Value**:
    * `dict[str, Any]`: A dictionary containing the result of the execution, and file field
        ```json
        {
            execution result,
            "files": [],  # kept for API backward-compatibility
        }
        ```
* **Side Effects**:
    * Produce `ExecutionStartedEvent(session_id)`
    * Calls kernel with given kernel id `execute`.
        * When kernel with kernel_id does not exists, `RuntimeError` raises
        * When asyncio.CancelledError occurs, `ExecutionCancelledEvent` is produced
    * If Kernel execution is "finished", `ExecutionFinishedEvent(session_id)` produced
    * If Kernel execution is "exec-timeout", `ExecutionTimeoutEvent(session_id)` produced and `inject_container_lifecycle_event` is called with `LifecycleEvent.DESTROY` and `EXEC_TIMEOUT`
---

## `trigger_batch_execution`
* **Description**: Initiates a batch execution task for a kernel.
* **Response Value**:
    * `None`.
* **Side Effects**:
    * Calls `self.agent.create_batch_execution_task()`, which likely sets up a background process or mechanism in the agent to feed code to the kernel for batch processing.

---

## `start_service`
* **Description**: Starts a specified service within a kernel's container.
* **Response Value**:
    * `dict[str, Any]`: Information about the started service (e.g., access points, status).
* **Side Effects**:
    * Calls `self.agent.start_service()`. This might involve:
        * Executing commands inside the container to launch the service.
        * Potentially exposing new ports or endpoints related to the service.

---

## `get_commit_status`
* **Description**: Checks the status of an ongoing or previous commit operation for a kernel.
* **Input Value**:
    * `kernel_id: KernelId`: Identifier for the kernel being used
    * `subdir: str`: path for organizing files or outputs
* **Response Value**:
    * `CommitStatus`
        * If image commit is ongoing(If lock_path(image-comit-path / subdir / lock / kernel_id) exists), response value is `CommitStatus.ONGOING`
        * If image commit is completed(If lock_path doesn't exit), response value is `CommitStatus.READY`
* **Side Effects**:
    * Calls kernel with `kernel_id` a `check_duplicate_commit(kernel_id, subdir)` method.
---

## `commit`
* **Description**: Commits the current state of a kernel's container (or a subdirectory within it) to a new image.
* **Input Value**: 
    * `reporter: ProgressReporter` : An instance of ProgressReporter to track and report progress(currently not used)
    * `kernel_id: KernelId`: Identifier for the kernel being used
    * `subdir: str`: path for organizing files or outputs
    * `canonical: str | None = None`: Optional canonical name or path for reference
    * `filename: str | None = None`: Optional filename to use for output or identification
    * `extra_labels: dict[str, str] = {}`: Additional labels as a dictionary
* **Response Value**:
    * `None`
* **Side Effects**:
    * Call kernel's `commit` method with same parameters
        * Creates necessary directories for the specified output path and lock_path
        * Commit new image with provided canonical and labels
        * If `filename` provided, the newly created Docker image is exported as a gzipped tarball to `path / filename`. After a successful export, this intermediate Docker image (created in the previous step) is deleted.
        * lock_path will be removed after all commit process

### Test Scenario 1 - Basic Commit without Export

* **Given**:
    * A kernel with `kernel_id_VALID` is running and its underlying container exists.
    * The Docker image `myimage:basic_commit` does not currently exist in the local Docker image list.
    * The directories `{base_commit_path}/test_commit_no_export/` and `{base_commit_path}/test_commit_no_export/lock/` may or may not exist on the host.
* **When**:
    * The `commit` method is called with the following input values:
        * `reporter`: A mock `ProgressReporter` instance.
        * `kernel_id`: `kernel_id_VALID`.
        * `subdir`: `"test_commit_no_export"`.
        * `canonical`: `"myimage"`.
        * `filename`: `None`.
        * `extra_labels`: `{"author": "test_user", "version": "1.0"}`.
* **Then**:
    * The method returns nothing
    * The directory `{base_commit_path}/test_commit_no_export/` is created on the host if it didn't exist.
    * The directory `{base_commit_path}/test_commit_no_export/lock/` is created on the host if it didn't exist.
    * A new Docker image tagged `myimage:latest` exists in the local Docker image list.
    * The created Docker image `myimage:latest` includes the labels `author="test_user"` and `version="1.0"`.
    * No gzipped tarball file is created on the host filesystem (because `filename` was `None`).
    * The Docker image `myimage:latest` is not deleted from the local Docker image list by this operation.


### Test Scenario 2 - Commit with Export

* **Given**:
    * An agent is running.
    * A kernel with `kernel_id_VALID` is running and its underlying container exists.
    * The Docker image `myimage:exported_commit` does not currently exist.
    * The file on the host filesystem at `{base_commit_path}/test_commit_with_export/exported_image.tar.gz` does not exist.
    * The directories `{base_commit_path}/test_commit_with_export/` and `{base_commit_path}/test_commit_with_export/lock/` may or may not exist.
* **When**:
    * The `commit` method is called with the following input values:
        * `reporter`: A mock `ProgressReporter` instance.
        * `kernel_id`: `kernel_id_VALID`.
        * `subdir`: `"test_commit_with_export"`.
        * `canonical`: `"myimage:exported_commit"`.
        * `filename`: `"exported_image"`.
        * `extra_labels`: `{"status": "exported"}`.
* **Then**:
    * The method returns nothing
    * The directory `{base_commit_path}/test_commit_with_export/` is created on the host if it didn't exist.
    * The directory `{base_commit_path}/test_commit_with_export/lock/` is created on the host if it didn't exist.
    * A lock file named `{kernel_id_VALID}` is created at the path `{base_commit_path}/test_commit_with_export/lock/{kernel_id_VALID}` on the host during the operation and is removed upon completion or failure.
    * A gzipped tarball file named `exported_image` exists on the host filesystem at the path `{base_commit_path}/test_commit_with_export/exported_image`.
    * The Docker image `myimage:exported_commit` no longer exists in the local Docker image list


### Test Scenario 3 - Commit with No Canonical Name

* **Given**:
    * An agent is running.
    * A kernel with `kernel_id_VALID` is running and its underlying container exists.
    * The directories `{base_commit_path}/test_commit_no_canonical/` and `{base_commit_path}/test_commit_no_canonical/lock/` may or may not exist.
* **When**:
    * The `commit` method is called with the following input values:
        * `reporter`: A mock `ProgressReporter` instance.
        * `kernel_id`: `kernel_id_VALID`.
        * `subdir`: `"test_commit_no_canonical"`.
        * `canonical`: `None`.
        * `filename`: `None`.
        * `extra_labels`: `{}`.
* **Then**:
    * The method returns nothing
    * The directory `{base_commit_path}/test_commit_no_canonical/` is created on the host if it didn't exist.
    * The directory `{base_commit_path}/test_commit_no_canonical/lock/` is created on the host if it didn't exist.
    * A lock file named `{kernel_id_VALID}` is created at `{base_commit_path}/test_commit_no_canonical/lock/{kernel_id_VALID}` and subsequently removed.
    * A new Docker image is created (its identifier, e.g., image ID, should be verifiable; specific tagging like `<none>:<none>` depends on Docker's default commit behavior).
    * No gzipped tarball file is created on the host filesystem.
    * The created Docker image is not deleted by this operation.


### Test Scenario 4 - Commit for a Non-Existent Kernel ID

* **Given**:
    * An agent is running.
    * A kernel with `kernel_id_INVALID` does not exist or is not in a running state.
    * The Docker image `myimage:fail_commit` does not exist.
* **When**:
    * The `commit` method is called with the following input values:
        * `reporter`: A mock `ProgressReporter` instance.
        * `kernel_id`: `kernel_id_INVALID`.
        * `subdir`: `"test_commit_invalid_kernel"`.
        * `canonical`: `"myimage:fail_commit"`.
        * `filename`: `None`.
        * `extra_labels`: `{}`.
* **Then**:
    * The method returns nothing
    * Raise `KeyError` as kernel are not found in `self.kernel_registry[kernel_id]`


### Test Scenario 5 - Concurrent Commit Attempts on the Same Kernel/Subdir

* **Given**:
    * An agent is running.
    * A kernel with `kernel_id_A` is running and its underlying container exists.
* **When**:
    * The `commit` method is called twice, nearly simultaneously, with the *same* `kernel_id_A` and `subdir_X`, but potentially different other parameters:
        * Call 1: `kernel_id_A`, `subdir_X`, `canonical="img1:tag1"`, `filename="f1.tar.gz"`
        * Call 2: `kernel_id_A`, `subdir_X`, `canonical="img2:tag2"`, `filename="f2.tar.gz"`
* **Then**:
    * One of the calls (e.g., Call 1) completes successfully.
    * The other call (e.g., Call 2) completes after call 1
    * Only one set of the expected successful side effects occurs:
        * EITHER a Docker image `img1:tag1` is temporarily created, file `f1.tar.gz` is exported to `{base_commit_path}/subdir_X/f1.tar.gz`, and image `img1:tag1` is deleted.
        * Then a Docker image `img2:tag2` is temporarily created, file `f2.tar.gz` is exported to `{base_commit_path}/subdir_X/f2.tar.gz`, and image `img2:tag2` is deleted.
        * Finally file `f1.tar.gz`, `f2.tar.gz` both exists


---

## `push_image`
* **Description**: Pushes a locally built or committed image to a specified container registry.
* **Input Value**:
    * `image_ref: ImageRef`: Reference to the container image
    * `registry_conf: ImageRegistry`: Configuration for the image registry
    * `timeout: float | None | Sentinel = Sentinel.TOKEN` : Timeout value in seconds, or a sentinel for not setting timeout
* **Response Value**:
    * `None`
* **Side Effects**:
    * If image_ref is local, agent do nothing
    * Push image by container runtime with auth cred created with registry username and password
    * If Image push fails, raise `RuntimeError`

### Test Scenario 1 - Successful Push with Authentication to Private Registry

* **Given**:
    * An `ImageRef` instance, `image_to_push_private`, is defined with:
        * `name="myimage"`
        * `project="myproject"`
        * `tag="latest"`
        * `registry="myprivatereg.example.com"`
        * `architecture="amd64"`
        * `is_local=False`
    * A local Docker image corresponding to the canonical name derived from `image_to_push_private` (e.g., `"myprivatereg.example.com/myproject/myimage:latest"`) exists in the agent's local Docker storage.
    * An `ImageRegistry` instance, `private_registry_config`, is defined as:
        * `name="myprivatereg.example.com"`
        * `url="https://myprivatereg.example.com"`
        * `username="valid_user"`
        * `password="valid_password"`
    * The image (derived from `image_to_push_private.canonical`) either does not yet exist in the remote private registry at `private_registry_config.url` or can be overwritten.
* **When**:
    * The `push_image` method is called with:
        * `image_ref`: `image_to_push_private`
        * `registry_conf`: `private_registry_config`
        * `timeout`: A reasonable value (e.g., `300.0`) or the default `Sentinel.TOKEN`.
* **Then**:
    * The method completes successfully and returns `None` (no exception is raised).
    * The Docker image (derived from `image_to_push_private.canonical`) is successfully pushed to and now exists in the remote private registry `myprivatereg.example.com`. (This would be verified by checking the remote registry).


### Test Scenario 2 - Attempt to Push an Image Marked as Local

* **Given**:
    * An `ImageRef` instance, `local_image_ref`, is defined with:
        * `name="mylocalimg"`
        * `project=None`
        * `tag="test"`
        * `registry="local"`
        * `architecture="amd64"`
        * `is_local=True`
* **When**:
    * The `push_image` method is called with:
        * `image_ref`: `local_image_ref`
        * `registry_conf`: `any_registry_config`
        * `timeout`: A reasonable value or the default.
* **Then**:
    * The method completes successfully and returns `None` immediately.
    * No attempt is made to push the image (derived from `local_image_ref.canonical`) to any remote registry.
    * No Docker push-related errors are logged or raised.


### Test Scenario 3 - Push Failure due to Invalid Authentication

* **Given**:
    * An `ImageRef` instance, `image_for_auth_fail`, is defined with:
        * `name="imageauthfail"`
        * `project="secureproject"`
        * `tag="1.0"`
        * `registry="myprivatereg.example.com"`
        * `architecture="amd64"`
        * `is_local=False`
    * A local Docker image corresponding to `image_for_auth_fail.canonical` exists in the agent's local Docker storage.
    * An `ImageRegistry` instance, `invalid_auth_registry_config`, is defined as:
        * `name="myprivatereg.example.com"`
        * `url="https://myprivatereg.example.com"`
        * `username="invalid_user"`
        * `password="wrong_password"`
* **When**:
    * The `push_image` method is called with:
        * `image_ref`: `image_for_auth_fail`
        * `registry_conf`: `invalid_auth_registry_config`
        * `timeout`: A reasonable value or the default.
* **Then**:
    * A `RuntimeError` is raised by the method.
    * The image (derived from `image_for_auth_fail.canonical`) is not pushed to the remote registry.


### Test Scenario 4 - Push Operation Timeout

* **Given**:
    * An `ImageRef` instance, `large_image_for_timeout`, is defined with:
        * `name="verylargeimage"`
        * `project="testproject"`
        * `tag="1.0"`
        * `registry="slowregistry.example.com"`
        * `architecture="amd64"`
        * `is_local=False`
    * A local Docker image corresponding to `large_image_for_timeout.canonical` exists
* **When**:
    * The `push_image` method is called with:
        * `image_ref`: `large_image_for_timeout`
        * `registry_conf`: `slow_registry_config`
        * `timeout`: A very short float value (e.g., `0.01` seconds) known to be less than the expected push completion time.
* **Then**:
    * A `RuntimeError` is raised by the method.
    * The image (derived from `large_image_for_timeout.canonical`) is not completely pushed or is not present in the remote registry.

---

## `purge_images`
* **Description**: Removes specified container images with `force` and `noprune` options. This method is usually called by `PurgeImageAction`, which is caused by Image GraphQL mutation
* **Input Value**:
    * `request: PurgeImagesReq`
        * `image_canonicals: list[str]` - List of image canonical names to be purged
        * `force: bool = False` - If true, forces removal of images even if they are in use
        * `noprune: bool = False` - If true, prevents removal of untagged parent images
* **Response Value**:
    * `PurgeImagesResp`: 
        * `image: str` - The canonical name of the image that was processed.
        * `error: Optional[str] = None` - Error message if deletion failed; otherwise, None
* **Side Effects**:
    * For each image specified in `request.images`:
        Attempts to delete the image using `docker.images.delete()` with `force` and `noprune` flags
    * If an image deletion fails, `PurgeImagesResp` returned with an `error` field filled

### Test Scenario 1 - Successfully Purge a Single Existing Image

* **Given**:
    * A Docker image named `"image-to-delete:v1"` exists in the local Docker storage.
* **When**:
    * The `purge_images` method is called with a `PurgeImagesReq` object where:
        * `images = ["image-to-delete:v1"]`
        * `force = False`
        * `noprune = False`
* **Then**:
    * The method returns a `PurgeImagesResp` object (let's call the returned instance `overall_response`).
    * `overall_response.responses` is a list containing one item.
    * The item in `overall_response.responses` has `image` field with value `"image-to-delete:v1"` and its `error` field is `None` (or not present, indicating success).
    * The Docker image `"image-to-delete:v1"` no longer exists in the local Docker storage.


### Test Scenario 2 - Successfully Purge Multiple Existing Images

* **Given**:
    * Docker images named `"image-a:latest"` and `"image-b:1.0"` exist in the local Docker storage.
* **When**:
    * The `purge_images` method is called with a `PurgeImagesReq` object where:
        * `images = ["image-a:latest", "image-b:1.0"]`
        * `force = False`
        * `noprune = False`
* **Then**:
    * The method returns a `PurgeImagesResp` object (`overall_response`).
    * `overall_response.responses` is a list containing two items.
    * Each item in `overall_response.responses` indicates success for its respective image (e.g., `image` field matches the input image name, `error` field is `None`).
    * The Docker images `"image-a:latest"` and `"image-b:1.0"` no longer exist in the local Docker storage.


### Test Scenario 3 - Attempt to Purge a Non-Existent Image

* **Given**:
    * A Docker image named `"ghost-image:v0"` does not exist in the local Docker storage.
* **When**:
    * The `purge_images` method is called with a `PurgeImagesReq` object where:
        * `images = ["ghost-image:v0"]`
        * `force = False`
        * `noprune = False`
* **Then**:
    * The method returns a `PurgeImagesResp` object (`overall_response`).
    * `overall_response.responses` is a list containing one item.
    * The item in `overall_response.responses` has `image = "ghost-image:v0"` and its `error` field contains an error message (e.g., indicating "No such image" or "image not found").
    * The local Docker storage state regarding `"ghost-image:v0"` remains unchanged (it still does not exist).


### Test Scenario 4 - Attempt to Purge an Image in Use (Without Force)

* **Given**:
    * A Docker image named `"busy-image:stable"` exists in the local Docker storage.
    * A container is currently running that was created from the `"busy-image:stable"` image.
* **When**:
    * The `purge_images` method is called with a `PurgeImagesReq` object where:
        * `images = ["busy-image:stable"]`
        * `force = False`
        * `noprune = False`
* **Then**:
    * The method returns a `PurgeImagesResp` object (`overall_response`).
    * `overall_response.responses` is a list containing one item.
    * The item in `overall_response.responses` has `image = "busy-image:stable"` and its `error` field contains an error message indicating the image is in use (ex. "conflict: unable to delete image...image is being used by running container...").
    * The Docker image `"busy-image:stable"` still exists in the local Docker storage.
    * The container using `"busy-image:stable"` is still running.


### Test Scenario 5 - Successfully Purge an Image in Use (With Force)

* **Given**:
    * A Docker image named `"busy-image:force-delete"` exists in the local Docker storage.
    * A container is currently running that was created from the `"busy-image:force-delete"` image.
* **When**:
    * The `purge_images` method is called with a `PurgeImagesReq` object where:
        * `images = ["busy-image:force-delete"]`
        * `force = True`
        * `noprune = False`
* **Then**:
    * The method returns a `PurgeImagesResp` object (`overall_response`).
    * `overall_response.responses` is a list containing one item.
    * The item in `overall_response.responses` has `image = "busy-image:force-delete"` and its `error` field is `None`.
    * The Docker image `"busy-image:force-delete"` no longer exists in the local Docker storage.
    * The container that was using `"busy-image:force-delete"` continues to run


### Test Scenario 6 - Purge a Mix of Existing, Non-Existent, and In-Use (Without Force) Images

* **Given**:
    * A Docker image named `"delete-me:ok"` exists in local Docker storage.
    * A Docker image named `"iamghost:hmm"` does not exist in local Docker storage.
    * A Docker image named `"dont-delete-me:busy"` exists and is used by a running container.
* **When**:
    * The `purge_images` method is called with a `PurgeImagesReq` object where:
        * `images = ["delete-me:ok", "iamghost:hmm", "dont-delete-me:busy"]`
        * `force = False`
        * `noprune = False`
* **Then**:
    * The method returns a `PurgeImagesResp` object (`overall_response`)
    * `overall_response.responses` is a list containing three items
    * The item for `"delete-me:ok"` indicates success (error is `None`)
    * The item for `"iamghost:hmm"` indicates failure with an error message
    * The item for `"dont-delete-me:busy"` indicates failure with an error message
    * The Docker image `"delete-me:ok"` no longer exists in local Docker storage
    * The Docker image `"dont-delete-me:busy"` still exists in local Docker storage


### Test Scenario 7 - Purge an Empty List of Images

* **Given**:
* **When**:
    * The `purge_images` method is called with a `PurgeImagesReq` object where:
        * `images = []`
        * `force = False`
        * `noprune = False`
* **Then**:
    * If an empty list (`[]`) is passed to the `images` field of `PurgeImagesReq`, values like `[None]` are not allowed, so a Pydantic validation error will occur.

---

## `shutdown_service`
* **Description**: Shuts down a running service within a kernel's container.
* **Input Value**:
    * `kernel_id: KernelId`: Id of the kernel to send message
    * `service: str`: Name of service
* **Response Value**:
    * `None`
* **Side Effects**:
    * Calls kernel with `kernel_id` a `shutdown_service(service)` method.
        * Inside kernel, it calls `self.runner.feed_shutdown_service(service)`

### Test Scenario 1 - Successfully Shut Down a Running Service

* **Given**:
    * An agent is running.
    * A kernel with `kernel_id_VALID` is active and its underlying container is running.
    * Inside the container for `kernel_id_VALID`, a service named `"active_service_A"` is currently running (e.g., its process exists, and/or it's listening on a specific port).
* **When**:
    * The `shutdown_service` RPC method is called on the agent with the following input values:
        * `kernel_id`: `kernel_id_VALID`
        * `service`: `"active_service_A"`
* **Then**:
    * The `shutdown_service` will return nothing
    * The service `"active_service_A"` inside the container associated with `kernel_id_VALID` is no longer running


### Test Scenario 2 - Attempt to Shut Down a Non-Existent Service in a Running Kernel

* **Given**:
    * An agent is running.
    * A kernel with `kernel_id_VALID` is active and its underlying container is running.
    * Inside the container for `kernel_id_VALID`, a service named `"ghost_service_B"` is *not* currently running.
* **When**:
    * The `shutdown_service` RPC method is called on the agent with the following input values:
        * `kernel_id`: `kernel_id_VALID`
        * `service`: `"ghost_service_B"`
* **Then**:
    * Returns Nothing
    * Exception log('unhandled exception while shutting down service app') will be last and do nothing


### Test Scenario 3 - Attempt to Shut Down a Service in a Non-Existent or Stopped Kernel

* **Given**:
    * An agent is running.
    * `kernel_id_INVALID` does not correspond to any active/running kernel known to the agent.
* **When**:
    * The `shutdown_service` RPC method is called on the agent with the following input values:
        * `kernel_id`: `kernel_id_INVALID`
        * `service`: `"any_service_C"`
* **Then**:
    * Returns Nothing
    * Exception log('unhandled exception while shutting down service app') will do nothing


### Test Scenario 4 - Attempt to Shut Down an Already Stopped Service

* **Given**:
    * An agent is running.
    * A kernel with `kernel_id_VALID` is active and its underlying container is running.
    * Inside the container for `kernel_id_VALID`, a service named `"already_stopped_service_D"` was previously running but has already been stopped.
* **When**:
    * The `shutdown_service` RPC method is called on the agent with the following input values:
        * `kernel_id`: `kernel_id_VALID`
        * `service`: `"already_stopped_service_D"`
* **Then**:
    * The service `"already_stopped_service_D"` inside the container for `kernel_id_VALID` remains in its stopped state.
    * No exceptions are raised


---

## `accept_file`
* **Description**: Uploads a file into a agent host machine. This method usually called due to `UploadFilesAction`, triggered by manager's session api `upload_files`
* **Input Value**:
    * `kernel_id: KernelId`: Id of the kernel to upload file
    * `filename: str`: Name of file
    * `filedata`: Binary data that wants to save
* **Response Value**:
    * `None`.
* **Side Effects**:
    * Calls kernel with `kernel_id` a `accept_file(filename, filedata)` method.
        * If user tries to upload file outside `/home/work` directory, `PermissionError` occurs
        * File is created under `{agent_config["container"]["scratch-root"]}/{kernel_id}/work/home/work/`
        * Parent directories for this target host path are created if they do not already exist
        * Raises a `RuntimeError` when if an `OSError` (e.g., disk full, host filesystem permission issues) occurs during the directory creation or file writing process on the host


### Test Scenario 1 - Successfully Upload a File to Top Level of Kernel's Work Directory

* **Given**:
    * A kernel with `kernel_id_VALID` is active and its corresponding scratch directory base (`{agent_config["container"]["scratch-root"]}/{kernel_id_VALID}/work/`) exists or can be created on the host.
    * The file `{agent_config["container"]["scratch-root"]}/{kernel_id_VALID}/work/uploaded_file.txt` does not currently exist on the host.
* **When**:
    * The `accept_file` method is called on the agent with:
        * `kernel_id`: `kernel_id_VALID`
        * `filename`: `"/home/work/uploaded_file.txt"` (Path inside the container's environment)
        * `filedata`: `b"This is test data."`
* **Then**:
    * The `accept_file` method returns `None`.
    * A file exists on the host filesystem at the path `{agent_config["container"]["scratch-root"]}/{kernel_id_VALID}/work/uploaded_file.txt`.
    * The content of the host file at that path is `b"This is test data."`.
    * No exceptions are raised by the agent's call.

---
### Test Scenario 2 - Attempt to Upload File Outside `/home/work` (e.g., using `../` in container path)

* **Given**:
    * A kernel with `kernel_id_VALID` is active.
* **When**:
    * The `accept_file` method is called on the agent with:
        * `kernel_id`: `kernel_id_VALID`
        * `filename`: `"/home/work/../attempt_outside.txt"` (Path attempting to traverse up from `/home/work`)
        * `filedata`: `b"Malicious data"`
* **Then**:
    * The `accept_file` method raises a `PermissionError`.
    * No file named `attempt_outside.txt` (or similar) is created anywhere outside the designated `{agent_config["container"]["scratch-root"]}/{kernel_id_VALID}/work/` directory on the host.

### Test Scenario 3 - Attempt to Upload File to an Absolute Path Outside `/home/work` (e.g., `/etc/`)

* **Given**:
    * A kernel with `kernel_id_VALID` is active.
* **When**:
    * The `accept_file` method is called on the agent with:
        * `kernel_id`: `kernel_id_VALID`
        * `filename`: `"/etc/new_file_by_agent"` (Absolute path outside the allowed container work directory)
        * `filedata`: `b"System data attempt"`
* **Then**:
    * The `accept_file` method raises a `PermissionError`.
    * No file is created at `/etc/new_file_by_agent` (or its host equivalent) on the host filesystem.

### Test Scenario 4 - Upload to a Non-Existent or Stopped Kernel

* **Given**:
    * `kernel_id_INVALID` does not correspond to any active/running kernel known to the agent.
* **When**:
    * The `accept_file` method is called on the agent with:
        * `kernel_id`: `kernel_id_INVALID`
        * `filename`: `"/home/work/anyfile.txt"`
        * `filedata`: `b"Orphaned data"`
* **Then**:
    * The `accept_file` method raises an `KeyError` exception

---

## `download_file`
* **Description**: Download a file in a agent host machine. This method usually called due to `DownloadFilesAction`, triggered by manager's session api `download_files`
* **Input Value**:
    * `kernel_id: KernelId`: Id of the kernel to download from
    * `filepath: str`: Path within `/home/work` to download file or directory
* **Response Value**:
    * The raw bytes of the tar archive containing the content at `/home/work/filepath`. 
        * If `/home/work/filepath` is a single file, the archive will contain that file
        * If it's a directory, the archive will contain the directory and its contents.
* **Side Effects**:
    * Calls kernel with `kernel_id` a `download_file(filepath)` method.
        * If user tries to download file outside `/home/work`, `PermissionError` occurs
        * If file is over 1 MiB, `ValueError` occurs
        * If unknown docker error occurs, `RuntimeError` raises

### Test Scenario 1 - Successfully Download a Single Existing File

* **Given**:
    * A kernel with `kernel_id_VALID` is active, and its underlying container is running.
    * On the host filesystem, a file exists at the path corresponding to what is `/home/work/data/sample.txt` inside the kernel's container (and in host filesystemt, `{scratch_root}/{kernel_id_VALID}/work/data/sample.txt`), and its content is `b"Test content for download."`.
* **When**:
    * The agent's `download_file` method (as called by the RPC) is invoked with:
        * `kernel_id`: `kernel_id_VALID`
        * `filepath`: `"/home/work/data/sample.txt"` (Path inside the container)
* **Then**:
    * The method returns a `bytes` object.
    * When the returned `bytes` object is interpreted as a tar archive, it contains a single entry (e.g., `sample.txt` or `data/sample.txt` depending on `get_archive` behavior for a specific file path) whose content is `b"Test content for download."`.
    * No exceptions are raised by the agent's method.


### Test Scenario 2 - Successfully Download an Existing Directory

* **Given**:
    * A kernel with `kernel_id_VALID` is active, and its container is running.
    * On the host filesystem, a directory exists at the path corresponding to `/home/work/project_files/` inside the kernel's container, containing (and in host filesystemt, `{scratch_root}/{kernel_id_VALID}/work/project_files/`):
        * `project_files/file1.py` (content: `b"print('hello')"`).
        * `project_files/docs/readme.md` (content: `b"# Project"`)
* **When**:
    * The agent's `download_file` method is invoked with:
        * `kernel_id`: `kernel_id_VALID`
        * `filepath`: `"/home/work/project_files"`
* **Then**:
    * The method returns a `bytes` object.
    * When the returned `bytes` object is interpreted as a tar archive, it contains entries corresponding to `project_files/file1.py` (with content `b"print('hello')"`) and `project_files/docs/readme.md` (with content `b"# Project"`).
    * No exceptions are raised by the agent's method.


### Test Scenario 3 - Attempt to Download File/Directory Outside `/home/work`

* **Given**:
    * A kernel with `kernel_id_VALID` is active and its container is running.
* **When**:
    * The agent's `download_file` method is invoked with:
        * `kernel_id`: `kernel_id_VALID`
        * `filepath`: `"/etc/important_config"` (Path outside the allowed `/home/work`)
* **Then**:
    * The agent's `download_file` method raises a `PermissionError`


### Test Scenario 4 - Attempt to Download Content Resulting in Tar Archive > 1 MiB

* **Given**:
    * A kernel with `kernel_id_VALID` is active, and its container is running.
    * A file or directory exists at `/home/work/very_large_data/` inside the kernel's container(and in host filesystemt, `{scratch_root}/{kernel_id_VALID}/work/very_large_data/`), such that its tar archive representation (as returned by Docker's `get_archive`) would exceed 1 MiB
* **When**:
    * The agent's `download_file` method is invoked with:
        * `kernel_id`: `kernel_id_VALID`
        * `filepath`: `"/home/work/very_large_data"`
* **Then**:
    * The agent's `download_file` method raises a `ValueError` with a message "Too large archive file exceeding 1 MiB".


### Test Scenario 5 - Attempt to Download a Non-Existent Path Within `/home/work`

* **Given**:
    * A kernel with `kernel_id_VALID` is active and its container is running.
    * The path `/home/work/non_existent_file.txt` does not correspond to any actual file or directory within the kernel's container environment.
* **When**:
    * The agent's `download_file` method is invoked with:
        * `kernel_id`: `kernel_id_VALID`
        * `filepath`: `"/home/work/non_existent_file.txt"`
* **Then**:
    * The agent's `download_file` method raises a `RuntimeError`


### Test Scenario 6 - Attempt to Download from a Non-Existent or Stopped Kernel

* **Given**:
    * `kernel_id_INVALID` does not correspond to any active or running kernel known to the agent.
* **When**:
    * The agent's `download_file` method is invoked with:
        * `kernel_id`: `kernel_id_INVALID`
        * `filepath`: `"/home/work/any_file.txt"`
* **Then**:
    * The agent's `download_file` method raises a `KeyError` exception

---

## `download_single`
* **Description**: Download a single file in a agent host machine. This method usually called due to `DownloadFilesAction`, triggered by manager's session api `download_files`
* **Input Value**:
    * `kernel_id: KernelId`: Id of the kernel to download file from
    * `filepath: str`: Path within `/home/work` to download file
* **Response Value**:
    * `bytes`: The raw content bytes of the single file specified by `/home/work/filepath`
* **Side Effects**:
    * Calls kernel with `kernel_id` a `download_single(filepath)` method.
        * If user tries to download file outsid `/home/work`, `PermissionError` occurs
        * If file is over 1 MiB, `ValueError` occurs
        * If the tar archive contains more than one entry (i.e., it's not a single file archive as expected), a `ValueError` raises
        * If the single file cannot be extracted or read from the archive (e.g., archive is malformed or the expected entry is not a readable file), a `ValueError` is raised
        * If unknown docker error occurs, `RuntimeError` raises

### Test Scenario 1 - Successfully Download a Single Existing File

* **Given**:
    * A kernel with `kernel_id_VALID` is active, and its underlying container is running.
    * On the host filesystem, a file exists at the path corresponding to `/home/work/documents/report.txt` inside the kernel's container (`{scratch_root}/{kernel_id_VALID}/work/documents/report.txt`), and its content is `b"This is the final report."`.
* **When**:
    * The agent's `download_single` method called with:
        * `kernel_id`: `kernel_id_VALID`
        * `filepath`: `"/home/work/documents/report.txt"` (Path inside the container)
* **Then**:
    * The method returns a `bytes` object.
    * The returned `bytes` object is equal to `b"This is the final report."`.
    * No exceptions are raised by the agent's method.


### Test Scenario 2 - Attempt to Download a Non-Empty Directory (Expected to Fail due to Multiple Entries in Tar)

* **Given**:
    * A kernel with `kernel_id_VALID` is active, and its container is running.
    * On the host filesystem, a directory exists at the path corresponding to `/home/work/my_pictures/` inside the kernel's container, and this directory contains one or more files (e.g., `my_pictures/photo1.jpg`, `my_pictures/photo2.png`).
* **When**:
    * The agent's `download_single` method is invoked with:
        * `kernel_id`: `kernel_id_VALID`
        * `filepath`: `"/home/work/my_pictures"`
* **Then**:
    * The agent's `download_single` method raises a `ValueError` with a message "Expected a single-file archive but found multiple files from /home/work/my_pictures".


### Test Scenario 3 - Attempt to Download File Outside `/home/work`

* **Given**:
    * A kernel with `kernel_id_VALID` is active and its container is running.
* **When**:
    * The agent's `download_single` method is invoked with:
        * `kernel_id`: `kernel_id_VALID`
        * `filepath`: `"/usr/bin/binary"` (Path outside the allowed `/home/work`)
* **Then**:
    * The agent's `download_single` method raises a `PermissionError` with a message similar to "You cannot download files outside /home/work".


### Test Scenario 4 - Attempt to Download a File Whose Tar Archive Exceeds 1 MiB

* **Given**:
    * A kernel with `kernel_id_VALID` is active.
    * A single file exists at `/home/work/data_archive_source.dat` inside the kernel's container. This file itself might be slightly less than 1MiB, but its representation within a tar archive (due to tar headers/metadata) causes the total tar archive size to exceed 1 MiB
* **When**:
    * The agent's `download_single` method is invoked with:
        * `kernel_id`: `kernel_id_VALID`
        * `filepath`: `"/home/work/data_archive_source.dat"`
* **Then**:
    * The agent's `download_single` method raises a `ValueError` with a message similar to "Too large archive file exceeding 1 MiB".


### Test Scenario 5 - Attempt to Download a Non-Existent File Within `/home/work`

* **Given**:
    * A kernel with `kernel_id_VALID` is active and its container is running.
    * The path `/home/work/fantasy_novel.txt` does not correspond to any actual file within the kernel's container environment.
* **When**:
    * The agent's `download_single` method is invoked with:
        * `kernel_id`: `kernel_id_VALID`
        * `filepath`: `"/home/work/fantasy_novel.txt"`
* **Then**:
    * The agent's `download_single` method raises a `RuntimeError`


### Test Scenario 6 - Attempt to Download from a Non-Existent or Stopped Kernel

* **Given**:
    * `kernel_id_GONE` does not correspond to any active or running kernel known to the agent.
* **When**:
    * The agent's `download_single` method is invoked with:
        * `kernel_id`: `kernel_id_GONE`
        * `filepath`: `"/home/work/any_single_file.dat"`
* **Then**:
    * The agent's `download_single` method raises a `KeyError` exception


---

## `list_files`
* **Description**: List files in a agent host machine. This method usually called due to `ListFilesAction`, triggered by manager's session api `list_files`
* **Input Value**:
    * `kernel_id: KernelId`: Id of the kernel to list files from
    * `filepath: str`: Path within `/home/work` to list files and directories
* **Response Value**:
    * A Json data
        * `"files": str` - A JSON string. When parsed, this string yields a list of objects, where each object represents a file or directory and contains details.
        * `"errors": str` - Any error output captured from the standard error stream of the docker exec command or the script executed within the container. This will be an empty string if no errors occurred.
        * `"abspath": str` - The original `file_path` argument that was passed to the method for listing.
* **Side Effects**:
    * Calls kernel with `kernel_id` a `list_files(path)` method.
        * If user tries to download file outsid `/home/work`, `PermissionError` occurs
        * Getting lists files and directories is achieved by executing a script inside the container, with running subprocess.

### Test Scenario 1 - Successfully List Files in an Existing, Non-Empty Directory

* **Given**:
    * A kernel with `kernel_id_VALID` is active, and its underlying container is running.
    * Inside the kernel's container, the directory `/home/work/documents/` exists and contains items such as:
        * `report.docx` (file)
        * `images` (subdirectory)
    * (These would physically be on the host at paths like `{scratch_root}/{kernel_id_VALID}/work/documents/report.docx`)
* **When**:
    * The agent's `list_files` method (as called by the RPC) is invoked with:
        * `kernel_id`: `kernel_id_VALID`
        * `path`: `"/home/work/documents"` (Path inside the container)
* **Then**:
    * The method returns a dictionary, let's call it `response_data`.
    * `response_data["abspath"]` is equal to `"/home/work/documents"`.
    * `response_data["errors"]` is an empty string.
    * `response_data["files"]` is a non-empty JSON string.
    * When the JSON string in `response_data["files"]` is parsed, it yields a list containing objects representing at least `report.docx` and `images`, each with correct metadata(ex. filename) 


### Test Scenario 2 - Successfully List an Existing Empty Directory

* **Given**:
    * A kernel with `kernel_id_VALID` is active, and its container is running.
    * Inside the kernel's container, the directory `/home/work/empty_dir/` exists and is empty.
    * (This directory would physically be on the host at `{scratch_root}/{kernel_id_VALID}/work/empty_dir/`)
* **When**:
    * The agent's `list_files` method is invoked with:
        * `kernel_id`: `kernel_id_VALID`
        * `path`: `"/home/work/empty_dir"`
* **Then**:
    * The method returns a dictionary `response_data`.
    * `response_data["abspath"]` is equal to `"/home/work/empty_dir"`.
    * `response_data["errors"]` is an empty string (or non-critical).
    * `response_data["files"]` is a JSON string that, when parsed, yields an empty list (`[]`).


### Test Scenario 3 - Attempt to List a Path Outside `/home/work`

* **Given**:
    * A kernel with `kernel_id_VALID` is active and its container is running.
* **When**:
    * The agent's `list_files` method is invoked with:
        * `kernel_id`: `kernel_id_VALID`
        * `path`: `"/etc"` (Path outside the allowed `/home/work`)
* **Then**:
    * The agent's `list_files` method raises a `PermissionError` with a message similar to "You cannot list files outside /home/work".


### Test Scenario 4 - Attempt to List a Non-Existent Path Within `/home/work`

* **Given**:
    * A kernel with `kernel_id_VALID` is active and its container is running.
    * The path `/home/work/this_folder_does_not_exist/` does not exist inside the kernel's container.
* **When**:
    * The agent's `list_files` method is invoked with:
        * `kernel_id`: `kernel_id_VALID`
        * `path`: `"/home/work/this_folder_does_not_exist"`
* **Then**:
    * The method returns a dictionary `response_data`.
    * `response_data["abspath"]` is equal to `"/home/work/this_folder_does_not_exist"`.
    * `response_data["files"]` is likely an empty string
    * `response_data["errors"]` contains a non-empty string, which includes an error message indicating the path was not found


### Test Scenario 5 - Attempt to List Files with an Invalid/Non-Existent Kernel ID

* **Given**:
    * `kernel_id_INVALID` does not correspond to any active or running kernel in the agent's `kernel_registry`.
* **When**:
    * The agent's `list_files` method is invoked with:
        * `kernel_id`: `kernel_id_INVALID`
        * `path`: `"/home/work/any_target_path"`
* **Then**:
    * The agent's `list_files` method raises a `KeyError`

---

## `shutdown`
* **Description**: Initiates the shutdown process for the agent.
* **Input Value**:
    * `stop_signal: signal.Signals`: Specifies the signal to use for agent shutdown (currently, only `signal.SIGTERM` is meaningful. Usually stop_signal value is signal.SIGTERM)
* **Response Value**:
    * `None`
* **Side Effects**:
    * Task Cancellation:
        * Cancels the agent's main socket communication task
        * Shuts down any implementation-specific periodic task groups (ex Docker-related ptask group)
        * Cancels all ongoing asynchronous batch execution tasks
        * Cancels all scheduled timer tasks.
        * Stops and cancels the Docker event monitoring task (if applicable)
    * Kernel and Container Lifecycle Management:
        * For every registered kernel:
            * Closes the kernel's runner
            * Calls the kernel object's own `close()` method for individual cleanup
        * Persists Kernel Registry: The current state of `self.kernel_registry` is serialized and written to a file with the name `{agent_config['container']['var-base-path']}/last_registry.{self.local_instance_id}.dat`
        * Conditional Full Kernel Destruction (if stop_signal is SIGTERM):
            * All registered kernels are explicitly signaled for destruction (`LifecycleEvent.DESTROY` with reason `AGENT_TERMINATION` is injected).
        * The shutdown process waits for these destruction operations to complete before proceeding. This ensures containers are removed
    * Event System and Handler Shutdown:
        * The container lifecycle event handler task is gracefully stopped.
        * An `AgentTerminatedEvent` with reason="shutdown" is produced
        * The event producer and event dispatcher components are closed.
    * External Service and Resource Cleanup:
        * Connection pools to external services (ex. Redis streams, Redis stats) are closed
        * Implementation-specific metadata server resources are cleaned up
        * The connection to the container engine (ex. Docker client) is closed
---

## `create_local_network`
* **Description**: Creates a container bridge network. Usually used when starting new session
* **Input Value**:
    * `network_name: str`: name of network that wants to make(Usually network name is composed of `bai-singlenode-{scheduled_session.id}`)
* **Response Value**:
    * `None`.
* **Side Effects**:
    * Bridge Network with given network_name, and label `"ai.backend.cluster-network": "1"`


### Test Scenario 1: Successfully Create a New Network

* **Given**:
    * An agent is running and has access to a Docker engine.
    * A Docker network named `"test-new-network-01"` does not currently exist in the Docker environment.
* **When**:
    * The agent's `create_local_network` method is called with:
        * `network_name`: `"test-new-network-01"`
* **Then**:
    * The `create_local_network` method returns `None`
    * A Docker network named `"test-new-network-01"` now exists in the Docker environment.
    * The created network `"test-new-network-01"` has the driver type `"bridge"`.
    * The created network `"test-new-network-01"` has the label `"ai.backend.cluster-network": "1"`.

### Test Scenario 2: Attempt to Create a Network That Already Exists

* **Given**:
    * An agent is running and has access to a Docker engine.
    * A Docker network named `"existing-network-02"` already exists in the Docker environment.
* **When**:
    * The agent's `create_local_network` method is called with:
        * `network_name`: `"existing-network-02"`
* **Then**:
    * The `create_local_network` method raises an exception
    * The state of the existing Docker network `"existing-network-02"` remains unchanged.

### Test Scenario 3: Attempt to Create a Network with an Invalid Name Format (Optional, based on Docker/aiodocker validation)

* **Given**:
    * An agent is running and has access to a Docker engine.
    * The string `"invalid@name#for_net"` is considered an invalid format for a Docker network name.
* **When**:
    * The agent's `create_local_network` method is called with:
        * `network_name`: `"invalid@name#for_net"`
* **Then**:
    * The `create_local_network` method raises an exception
    * No new network with the name `"invalid@name#for_net"` is created in the Docker environment.

---

## `destroy_local_network`
* **Description**: Destroys container network. Usually used when clean up a session
* **Input value**:
    * `network_name: str`: network name wants to destroy
* **Response Value**:
    * `None`.
* **Side Effects**:
    * Destroy container network name with given parameter

### Test Scenario 1: Successfully Destroy an Existing, Unused Network

* **Given**:
    * An agent is running and has access to a Docker engine.
    * A Docker network named `"delete-me-network-A"` exists in the Docker environment.
    * No running containers are currently connected to the `"delete-me-network-A"` network.
* **When**:
    * The agent's `destroy_local_network` method is called with:
        * `network_name`: `"delete-me-network-A"`
* **Then**:
    * The `destroy_local_network` method returns `None`
    * The Docker network `"delete-me-network-A"` no longer exists in the Docker environment.

### Test Scenario 2: Attempt to Destroy a Non-Existent Network

* **Given**:
    * An agent is running and has access to a Docker engine.
    * A Docker network named `"non-existent-network-B"` does not exist in the Docker environment.
* **When**:
    * The agent's `destroy_local_network` method is called with:
        * `network_name`: `"non-existent-network-B"`
* **Then**:
    * The `destroy_local_network` method raises an exception

### Test Scenario 3: Attempt to Destroy a Network Currently in Use by a Container

* **Given**:
    * An agent is running and has access to a Docker engine.
    * A Docker network named `"busy-network-C"` exists in the Docker environment.
    * At least one active container is connected to the `"busy-network-C"` network.
* **When**:
    * The agent's `destroy_local_network` method is called with:
        * `network_name`: `"busy-network-C"`
* **Then**:
    * The `destroy_local_network` method raises an exception
    * The Docker network `"busy-network-C"` still exists in the Docker environment.
    * The container(s) connected to `"busy-network-C"` remain running and connected.
