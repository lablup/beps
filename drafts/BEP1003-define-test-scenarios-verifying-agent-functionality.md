---
Author: Bokeum Kim (bkkim@lablup.com)
Status: Draft
---

## Agent RPC Operations

---

### `update_scaling_group`
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

### `ping`
* **Description**: A simple ping RPC to check connectivity.
* **Input Value**: 
    * `msg: str`: ping message
* **Response Value**:
    * `str`: The original message sent in the request.

---

### `gather_hwinfo`
* **Description**: Collects hardware metadata from the agent's host.
* **Response Value**:
    * `Mapping[str, HardwareMetadata]`: A dictionary containing various hardware details (e.g., CPU, GPU, memory).

---

### `ping_kernel`
* **Description**: Pings a specific running kernel to check its responsiveness.
* **Input Value**:
    * `kernel_id: str`: Id of kernel that wants to send ping
* **Response Value**:
    * `dict[str, float] | None`: A dictionary with timing/status information if the kernel responds, otherwise `None`.

---

### `sync_kernel_registry`
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

### `check_and_pull`
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

### `create_kernels`
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

### `destroy_kernel`
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

### `interrupt_kernel`
* **Description**: Sends an interrupt signal to a running kernel.
* **Input Value**: 
    * `kernel_id: str`
* **Response Value**:
    * `None`.
* **Side Effects**:
    * Calls `self.agent.interrupt_kernel()`, sending a interrup msg to kernel

---

### `get_completions`
* **Description**: Requests code completions from a kernel.
* **Input Value**: 
    * `kernel_id: str`
    * `text: str`
    * `opts: dict`
* **Response Value**:
    * JSON-parsed completion results containing suggested code completions

---

### `get_logs`
* **Description**: Retrieves logs from a specific kernel's container.
* **Input Value**: 
    * `kernel_id: str`
* **Response Value**:
    * JSON-parsed result of kernel logs

---

### `restart_kernel`
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

### `execute`
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

### `trigger_batch_execution`
* **Description**: Initiates a batch execution task for a kernel.
* **Response Value**:
    * `None`.
* **Side Effects**:
    * Calls `self.agent.create_batch_execution_task()`, which likely sets up a background process or mechanism in the agent to feed code to the kernel for batch processing.

---

### `start_service`
* **Description**: Starts a specified service within a kernel's container.
* **Response Value**:
    * `dict[str, Any]`: Information about the started service (e.g., access points, status).
* **Side Effects**:
    * Calls `self.agent.start_service()`. This might involve:
        * Executing commands inside the container to launch the service.
        * Potentially exposing new ports or endpoints related to the service.

---

### `get_commit_status`
* **Description**: Checks the status of an ongoing or previous commit operation for a kernel.
* **Input Value**:
    * `kernel_id: KernelId`
    * `subdir: str`
* **Response Value**:
    * `CommitStatus`
        * If image commit is ongoing(If lock_path(image-comit-path / subdir / lock / kernel_id) exists), response value is `CommitStatus.ONGOING`
        * If image commit is completed(If lock_path doesn't exit), response value is `CommitStatus.READY`
* **Side Effects**:
    * Calls kernel with `kernel_id` a `check_duplicate_commit(kernel_id, subdir)` method.
---

### `commit`
* **Description**: Commits the current state of a kernel's container (or a subdirectory within it) to a new image.
* **Input Value**: 
    * `kernel_id`
    * `subdir`
    * `canonical: str | None = None`
    * `filename: str | None = None`
    * `extra_labels: dict[str, str] = {}`
* **Response Value**:
    * `dict[str, Any]`: Contains a `bgtask_id` for the background commit operation, the `kernel_id`, and the `path` (if a filename is specified).
* **Side Effects**:
    * Call kernel's `commit` method with same parameters
        * Creates necessary directories for the specified output path and lock_path
        * Commit new image with provided canonical and labels
        * If `filename` provided, the newly created Docker image is exported as a gzipped tarball to `path / filename`. After a successful export, this intermediate Docker image (created in the previous step) is deleted.
        * lock_path will be removed after all commit process

---

### `push_image`
* **Description**: Pushes a locally built or committed image to a specified container registry.
* **Input Value**:
    * `image_ref: ImageRef`
    * `registry_conf: ImageRegistry`
    * `timeout: float | None | Sentinel = Sentinel.TOKEN``
* **Response Value**:
    * `None`
* **Side Effects**:
    * If image_ref is local, agent do nothing
    * Push image by container runtime with auth cred created with registry username and password
    * If Image push fails, raise `RuntimeError`

---

### `purge_images`
* **Description**: Removes specified local container images from the agent's host.
* **Input Value**:
    * `requset: PurgeImagesReq`
        * `image_canonicals: list[str]`
        * `force: bool`
        * `noprune: bool`
* **Response Value**:
    * `PurgeImagesResp`: 
        * `image: str`
        * `error: Optional[str] = None`
* **Side Effects**:
    * For each image specified in `request.images`:
        Attempts to delete the image using `docker.images.delete()` with `force` and `noprune` flags
    * If an image deletion fails, `PurgeImagesResp` returned with an `error` field filled

---

### `shutdown_service`
* **Description**: Shuts down a running service within a kernel's container.
* **Input Value**:
    * `kernel_id: KernelId`
    * `service: str`
* **Response Value**:
    * `None`
* **Side Effects**:
    * Calls kernel with `kernel_id` a `shutdown_service(service)` method.
        * Inside kernel, it calls `self.runner.feed_shutdown_service(service)`

---

### `accept_file`
* **Description**: Uploads a file into a kernel's environment.
* **Input Value**:
    * `kernel_id: KernelId`
    * `filename: str`
    * `filedata`
* **Response Value**:
    * `None`.
* **Side Effects**:
    * Calls kernel with `kernel_id` a `accept_file(filename, filedata)` method.
        * If user tries to download file outside `/home/work`, `PermissionError` occurs
        * File is created under `{agent_config["container"]["scratch-root"]}/{kernel_id}/work/home/work/`
        * Parent directories for this target host path are created if they do not already exist
        * Raises a `RuntimeError` when if an `OSError` (e.g., disk full, host filesystem permission issues) occurs during the directory creation or file writing process on the host


---

### `download_file`
* **Description**: Retrieves a specified path (file or directory) from within the container's /home/work directory as a tar archive.
* **Input Value**:
    * `kernel_id: KernelId`
    * `filepath: str`
* **Response Value**:
    * The raw bytes of the tar archive containing the content at `/home/work/filepath`. 
        * If `/home/work/filepath` is a single file, the archive will contain that file
        * If it's a directory, the archive will contain the directory and its contents.
* **Side Effects**:
    * Calls kernel with `kernel_id` a `download_file(filepath)` method.
        * If user tries to download file outside `/home/work`, `PermissionError` occurs
        * If file is over 1 MiB, `ValueError` occurs
        * If unknown docker error occurs, `RuntimeError` raises
---

### `download_single`
* **Description**: DRetrieves the content of a single file from within the container's /home/work directory.
* **Input Value**:
    * `kernel_id: KernelId`
    * `filepath: str`
* **Response Value**:
    * `bytes`: The raw content bytes of the single file specified by `/home/work/filepath`
* **Side Effects**:
    * Calls kernel with `kernel_id` a `download_single(filepath)` method.
        * If user tries to download file outsid `/home/work`, `PermissionError` occurs
        * If file is over 1 MiB, `ValueError` occurs
        * If the tar archive contains more than one entry (i.e., it's not a single file archive as expected), a `ValueError` raises
        * If the single file cannot be extracted or read from the archive (e.g., archive is malformed or the expected entry is not a readable file), a `ValueError` is raised
        * If unknown docker error occurs, `RuntimeError` raises

---

### `list_files`
* **Description**: Lists files and directories at a given path within the kernel's /home/work directory
* **Input Value**:
    * `kernel_id: KernelId`
    * `filepath: str`
* **Response Value**:
    * A Json data
        * `"files": str` - A JSON string. When parsed, this string yields a list of objects, where each object represents a file or directory and contains details.
        * `"errors": str` - Any error output captured from the standard error stream of the docker exec command or the script executed within the container. This will be an empty string if no errors occurred.
        * `"abspath": str` - The original `file_path` argument that was passed to the method for listing.
* **Side Effects**:
    * Calls kernel with `kernel_id` a `list_files(path)` method.
        * If user tries to download file outsid `/home/work`, `PermissionError` occurs
        * Getting lists files and directories is achieved by executing a script inside the container, with running subprocess.

---

### `shutdown`
* **Description**: Initiates the shutdown process for the agent.
* **Input Value**:
    * `stop_signal: signal.Signals`
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
            * All registered kernels are explicitly signaled for destruction (`LifecycleEvent.DESTROY` with reason `AGENT_TERMINATION`).
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

### `create_local_network`
* **Description**: Creates a container bridge network
* **Input Value**:
    * `network_name: str`: name of network that wants to make
* **Response Value**:
    * `None`.
* **Side Effects**:
    * Bridge Network with given network_name, and label `"ai.backend.cluster-network": "1"`

---

### `destroy_local_network`
* **Description**: Destroys container network
* **Input value**:
    * `network_name: str`: network name wants to destroy
* **Response Value**:
    * `None`.
* **Side Effects**:
    * Destroy container network name with given parameter
