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
* **Response Value**:
    * `dict[str, Any]`: Contains the `kernel_id` and the `status` (e.g., "pending", "completed", "failed") of the commit.
* **Side Effects**:
    * Calls `self.agent.get_commit_status()`.

---

### `commit`
* **Description**: Commits the current state of a kernel's container (or a subdirectory within it) to a new image.
* **Response Value**:
    * `dict[str, Any]`: Contains a `bgtask_id` for the background commit operation, the `kernel_id`, and the `path` (if a filename is specified).
* **Side Effects**:
    * Starts a background task (`_commit`) via `self.agent.background_task_manager`.
    * The `_commit` task calls `self.agent.commit()`, which involves:
        * **Image Creation**: Creating a new container image from the state of the running kernel's container.
        * May produce events related to the commit progress/completion.

---

### `push_image`
* **Description**: Pushes a locally built or committed image to a specified container registry.
* **Response Value**:
    * `dict[str, Any]`: Contains a `bgtask_id` for the background push operation and the `canonical` name of the image being pushed.
* **Side Effects**:
    * Starts a background task (`_push_image`) via `self.agent.background_task_manager`.
    * The `_push_image` task calls `self.agent.push_image()`, which involves:
        * **Image Push**: Uploading the image layers to the target registry.
        * May produce events related to the push progress/completion.

---

### `purge_images` (RPCFunctionRegistryV2)
* **Description**: Removes specified local container images from the agent's host.
* **Response Value**:
    * `PurgeImagesResp`: An object detailing the results of the purge operation (e.g., which images were successfully deleted, any errors).
* **Side Effects**:
    * Calls `self.agent.purge_images()`, which involves:
        * **Image Deletion**: Removing the specified images from the local image store.
        * May produce events related to image deletion.

---

### `get_local_config`
* **Description**: Retrieves parts of the agent's local configuration.
* **Response Value**:
    * `Mapping[str, Any]`: A dictionary containing selected agent and watcher configurations (e.g., `abuse-report-path`).
* **Side Effects**:
    * Reads its own in-memory `local_config`.

---

### `shutdown_service`
* **Description**: Shuts down a running service within a kernel's container.
* **Response Value**:
    * The result from `self.agent.shutdown_service()` (the exact type is not specified in the RPC signature, could be status or `None`).
* **Side Effects**:
    * Calls `self.agent.shutdown_service()`. This might involve sending signals or commands to the service process within the container to terminate it.

---

### `upload_file`
* **Description**: Uploads a file into a kernel's environment.
* **Response Value**:
    * `None`.
* **Side Effects**:
    * Calls `self.agent.accept_file()`, which writes the provided `filedata` to the specified `filename` within the kernel's filesystem (potentially in a mounted volume or the container's writable layer).

---

### `download_file`
* **Description**: Downloads a file or directory (potentially archived) from a kernel's environment.
* **Response Value**:
    * The file data (e.g., `bytes` or a stream, depending on `self.agent.download_file()` implementation).
* **Side Effects**:
    * Calls `self.agent.download_file()` to read the file/directory from the kernel's filesystem.

---

### `download_single`
* **Description**: Downloads a single file from a kernel's environment.
* **Response Value**:
    * The file data (e.g., `bytes` or a stream, depending on `self.agent.download_single()` implementation).
* **Side Effects**:
    * Calls `self.agent.download_single()` to read the specified file from the kernel's filesystem.

---

### `list_files`
* **Description**: Lists files and directories at a given path within a kernel's environment.
* **Response Value**:
    * A list of file/directory entries (the exact structure depends on `self.agent.list_files()` implementation).
* **Side Effects**:
    * Calls `self.agent.list_files()` to inspect the filesystem within the kernel's environment.

---

### `shutdown_agent`
* **Description**: Initiates the shutdown process for the agent.
* **Response Value**:
    * `None` (currently, as the implementation is `pass`).
* **Side Effects**:
    * **TODO**: The implementation is noted as a TODO. A full implementation would likely:
        * Terminate all running kernels (**Container Deletion** for all kernels).
        * Clean up any other resources.
        * Stop the agent process itself.

---

### `create_local_network`
* **Description**: Creates a local (e.g., Docker) network on the agent's host.
* **Response Value**:
    * `None`.
* **Side Effects**:
    * Calls `self.agent.create_local_network()` to interact with the container runtime to create the network.

---

### `destroy_local_network`
* **Description**: Destroys a local network on the agent's host.
* **Response Value**:
    * `None`.
* **Side Effects**:
    * Calls `self.agent.destroy_local_network()` to interact with the container runtime to remove the network.

---

### `reset_agent`
* **Description**: Resets the agent by destroying all currently running kernels.
* **Response Value**:
    * `None` (after all kernel destruction tasks are gathered).
* **Side Effects**:
    * Iterates through all kernels in `self.agent.kernel_registry`.
    * For each kernel:
        * Schedules `self.agent.destroy_kernel()` which leads to:
            * **Container Deletion**.
            * **Event Production** (e.g., `KernelDestroyedEvent`).
        * Exceptions during individual kernel destruction are captured by `self.error_monitor` and logged.

---

### `assign_port`
* **Description**: Assigns an available port from the agent's pool.
* **Response Value**:
    * `int`: An available port number.
* **Side Effects**:
    * Removes a port from `self.agent.port_pool`.

---

### `release_port`
* **Description**: Releases a previously assigned port back to the agent's pool.
* **Response Value**:
    * `None`.
* **Side Effects**:
    * Adds the `port_no` back to `self.agent.port_pool`.

---

### `scan_gpu_alloc_map`
* **Description**: Scans and returns the current GPU allocation map based on running kernels.
* **Response Value**:
    * `Mapping[str, Any]`: A dictionary mapping identifiers (likely GPU IDs or kernel IDs) to allocation information.
* **Side Effects**:
    * Calls an external utility `scan_gpu_alloc_map` which likely inspects system state or container configurations to determine GPU usage by kernels.

---