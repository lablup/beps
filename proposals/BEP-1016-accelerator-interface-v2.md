---
Author: Joongi Kim (joongi@lablup.com)
Status: Draft
Created: 2025-11-28
---

# Accelerator Interface v2

## Prior Design

### `AbstractComputeDevice` API

| Function                                                         | Role                                                                                                                                   |
| ---------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| `list_devices()`                                                 | List the available devices in the node                                                                                                 |
| `available_slots()`                                              | List the currently available resource slot types as configured                                                                         |
| `get_metadata()`                                                 | Return the resource slot display metadata (human readable naming, etc.)                                                                |
| `extra_info()`                                                   | Return driver / compute API versions                                                                                                   |
| `get_node_hwinfo()`                                              | Return the node's hardware-specific information as an arbitrary key-value map                                                          |
| `create_alloc_map()`                                             | Create an `AbstractAllocMap` which may be provided by either the Backend.AI agent (discrete / fractional) or the plugin, as configured |
| `get_hooks(distro, arch)`                                        | Get additional host files to mount as library hooks depending on the given container image base distro information                     |
| `generate_docker_args(docker, device_alloc)`                     | Generate a nested dict to merge with the Docker container creation API params                                                          |
| `get_attached_devices(device_alloc)`                             | Extract the list of devices used in the given allocation, with their metadata                                                          |
| `get_docker_networks(device_alloc)`                              | Generate a list of Docker networks to attach depending on the given allocation                                                         |
| `generate_mounts(source_path, device_alloc)`                     | Generate additiona host files to mount                                                                                                 |
| `generate_resource_data(device_alloc)`                           | Generate a list of strings (in KEY=VALUE form) to put into `resource.txt` in the container                                             |
| `restore_from_container(container, alloc_map)`                   | Reconstruct the device allocation from the `aiodocker.DockerContainer` object                                                          |
| `gather_{node,container,process}_metrics(stat_ctx[, target_id])` | Collects the raw metric values such as processor and memory utilization per node, container, or process                                |

### Device Info

See BEP-1000 for the new proposal.

## Proposed Design

### Key Goals

* Make it applicable to non-Docker agent backends
    - Many existing plugin APIs are highly coupled with Docker-specific terminology and API parameter formats
* Allow programmatic extension of container lifecycle events 
    - e.g., Interact with a vendor-provided device management service when creating or destroying new containers in a node
* Tidy up redundant and messy methods that only expose partial information
* Provide more detailed accelerator metadata ([BEP-1000](https://github.com/lablup/beps/blob/main/proposals/BEP-1000-redefining-accelerator-metadata.md))

### `AbstractComputeDevice` API

| Function                                                         | Role                                                                                                               |
| ---------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| `list_devices()`                                                 | List the available devices in the node                                                                             |
| `configurable_slots()` ✨                                         | List the all possible resource slot types along with the display metadata                                          |
| `available_slots()` ✨                                            | List the currently allocatable resource slot types as configured                                                   |
| `create_alloc_map()`                                             | Create an `AbstractAllocMap` instance as configured                                                                |
| `create_lifecycle_hook(container, device_alloc)` ✨               | Create an `AbstractLifecycleHook` instance                                                                         |
| `alloc_to_devices(device_alloc)` ♻️                              | Extract the list of devices used in the given allocation, with their metadata                                      |
| `gather_{node,container,process}_metrics(stat_ctx[, target_id])` | Collects the raw metric values such as processor and memory utilization per node, container, or process            |
| `get_node_info()` ♻                                              | Get the node information such as driver/runtime versions and additional hardware info using a structured dataclass |

### `AbstractLifecycleHook` API ✨

| Function                            | Role                                                                                                                                                                                                                                              |
| ----------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `__init__(container, device_alloc)` | Initialize the instance with the given container and allocation                                                                                                                                                                                   |
| `pre_create()`                      | Invoked before container is created.<br>It may deny or (temporarily) fail the creation by raising predefined exceptions.<br>Should return container mounts, resource data (`resource.txt` lines), environment variables, and network attachments. |
| `post_create()`                     | Invoked after container is created.                                                                                                                                                                                                               |
| `pre_terminate()`                   | Invoked before container is terminated.<br>It cannot cancel the termination but may defer termination for plugin-specific cleanup.                                                                                                                |
| `post_terminate()`                  | Invoked after container is terminated.                                                                                                                                                                                                            |

This new API merges and replaces Docker-specific argument/mount generation methods in the prior design.

`AbstractLifecycleHook` should be designed as stateless, and it should be able to restore additional state from the container if necessary, to ensure that the Backend.AI Agent is fully restartable at any time.

### Discussion

* How to handle & distinguish in-place restarts and relocated restarts in lifecycle hooks?
* Would it better to provide a managed state-store interface to the lifecycle hook instances instead of requiring them to be stateless?
