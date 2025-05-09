----
Author: Joongi Kim (joongi@lablup.com)
Status: Draft
----

# Redefining Accelerator Metadata

## Current structure

```python
# ai.backend.common.types
# -----------------------
class ResourceSlotReprFormat(TypedDict):
    # I'd propose to rename this from AcceleratorNumberFormat.
    binary: bool
    round_length: int


class AcceleratorMetadata(TypedDict):
    slot_name: str
    description: str
    human_readable_name: str
    display_unit: str
    display_icon: str
    number_format: ResourceSlotReprFormat


# ai.backend.agent.resources
# --------------------------
class AbstractComputeDevice:
    device_id: DeviceId
    hw_location: str  # either PCI bus ID or arbitrary string
    memory_size: int  # bytes of available per-accelerator memory
    processing_units: int  # number of processing units (e.g., cores, SMP)
    numa_node: Optional[int]  # NUMA node ID (None if not applicable)
    
    ...

class AbstractComputePlugin(AbstractPlugin, metaclass=ABCMeta):
    key: DeviceName = DeviceName("accelerator")
    slot_types: Sequence[tuple[SlotName, SlotTypes]]
    exclusive_slot_types: set[str]

    @abstractmethod
    def get_metadata(self) -> AcceleratorMetadata: ...
    
    @abstractmethod
    async def list_devices(self) -> Collection[AbstractComputeDevice]: ...

    @abstractmethod
    async def available_slots(self) -> Mapping[SlotName, Decimal]: ...

    @abstractmethod
    def get_version(self) -> str: ...

    @abstractmethod
    async def extra_info(self) -> Mapping[str, str]: ...

    ...
```

### Problems

A single plugin may return only one AcceleratorMetadata.Example) If the cuda plugin has multiple devices with different configurations (e.g., 2 GPUs with MIG enabled and 6 GPUs without it), it cannot return two different AcceleratorMetadata instances.
It is confusing whether AcceleratorMetadata represents a resource slot or a device (it's for resource slot!). The name should be clarified.

We need to expand the metadata format to include various device capabilities such as compute precision support, partition capability, and memory hierarchy.

## Proposed structure (WIP)

```python
# ai.backend.common.types
# -----------------------
class ComputePrecisionSupportLevel(enum.StrEnum):
    NATIVE = enum.auto()
    EMULATED = enum.auto()
    NONE = enum.auto()


@attrs.define(auto_attribs=True)
class ComputePrecisionSupport:
    format: str  # e.g., "FP32", "TF32", "BF16", "FP16", "FP8", "INT8", "INT4", "FP4"
    support: ComputePrecisionSupportLevel


@attrs.define(auto_attribs=True)
class ComputeUnit:
    name: str  # e.g., "SM", "Tensor Core", "GPC", "CU", "TPC", "AI Core"
    count: int  # Number of such units in the device


@attrs.define(auto_attribs=True)
class MemorySpec:
    total_memory: int  # Total on-device memory capacity in bytes
    memory_type: str   # e.g., "HBM2", "HBM3", "GDDR6", "DDR5"
    bandwidth: Optional[float] = None          # Peak memory bandwidth in GiB/s
    l1_cache_per_core: Optional[int] = None    # L1 cache size per compute unit (bytes)
    l2_cache: Optional[int] = None             # L2 cache size (bytes, chip-wide or per die)
    shared_mem_per_core: Optional[int] = None  # Size of programmable shared memory per core (bytes)
    memory_channels: Optional[int] = None      # Number of memory channels or HBM stacks


@attrs.define(auto_attribs=True)
class PartitioningCapability:
    max_partitions: int = 1           # Maximum number of slices/partitions
    technology: Optional[str] = None  # Name of partitioning method (e.g., "MIG", "fGPU")


# ai.backend.agent.resources
# --------------------------
class AbstractComputeDevice:
    device_id: DeviceId
    hw_location: str          # either PCI bus ID or arbitrary string
    numa_node: Optional[int]  # NUMA node ID (None if not applicable)    
    memory_size: int          # (legacy-compat) size of the on-device memory (bytes)
    processing_units: int     # (legacy-compat) number of the representative compute units
    memory_spec: MemorySpec   # (new) detailed memory specification
    compute_units: Sequence[ComputeUnit]  # (new) detailed compute unit specification
    precision_support: Sequence[ComputePrecisionSupport]
    partioning_support: Sequence[PartitioningCapability]
    
    ...

class AbstractComputePlugin(AbstractPlugin, metaclass=ABCMeta):
    key: DeviceName = DeviceName("accelerator")
    slot_types: Sequence[tuple[SlotName, SlotTypes]]
    exclusive_slot_types: set[str]

    # CHANGED: now returns a list of AcceleratorMetadata
    @abstractmethod
    def get_metadata(self) -> Sequence[AcceleratorMetadata]: ...
    
    @abstractmethod
    async def list_devices(self) -> Collection[AbstractComputeDevice]: ...

    @abstractmethod
    async def available_slots(self) -> Mapping[SlotName, Decimal]: ...

    @abstractmethod
    def get_version(self) -> str: ...

    @abstractmethod
    async def extra_info(self) -> Mapping[str, str]: ...

    ...
```

## Goals

Generalize it for heterogeneous accelerators with multiple different partitioning support
Support multiple different resource slot definitions from a single plugin instance (e.g,. MIG + fGPU mixed in a single node).
Make it extensible without changing Python plugin interfaces, by using a comprehensive schema-based metadata interface using JSON.Consider defining an explicit jsonschema for easier validation.
Unify a-little-bit duplicated interfaces, like: "get_metadata()" and "extra_info()".
Potential impacts to fellow developers

The plugin interface update requires updates of the existing plugins.Instead of replacing existing interface, we could add another interface to support multiple interfaces at the same time.

