## Device Plugins: Why GPUs Aren't Like CPUs

In Kubernetes, CPU and memory are built-in, first-class resources. The kubelet understands them natively — it can slice, share, and account for them without any extra machinery. That's why your bin-packing story works so well for stateless services.

GPUs are different. They're external resources that Kubernetes doesn't inherently understand. The Device Plugin framework (introduced as a beta in k8s 1.10) is the bridge that was built to solve this. Here's how it works:

Each vendor (NVIDIA, AMD, Intel) writes a device plugin — a gRPC server that runs as a DaemonSet on each node. It registers with the kubelet via a Unix socket at /var/lib/kubelet/device-plugins/ and does three things: 

1. It advertises available devices (e.g., "this node has 4 nvidia.com/gpu")
2. It handles allocation when a pod requests one
3. It performs any device-specific setup (mounting device files, setting env vars, injecting volume mounts).

## Relationship between the Host and the Device.

- The Host: This is your CPU. It runs the operating system and executes your code. The Host is the commander; it’s in charge of the overall logic and tells the Device what to do.
- The Device: This is your GPU. It’s a powerful but specialized coprocessor designed for massively parallel computations. The Device is the accelerator; it doesn’t do anything until the Host gives it a task.

Your program always starts on the CPU. When you want the GPU to perform a task, like multiplying two large matrices, the CPU sends the instructions and the data over to the GPU.

## The CPU-GPU Interaction

The Host talks to the Device through a queuing system.

- CPU Initiates Commands: Your script, running on the CPU, encounters a line of code intended for the GPU (e.g., tensor.to('cuda')).
- Commands are Queued: The CPU doesn’t wait. It simply places this command onto a special to-do list for the GPU called a CUDA Stream.
- Asynchronous Execution: The CPU does not wait for the actual operation to be completed by the GPU, the host moves on to the next line of your script. This is called asynchronous execution, and it’s a key to achieving high performance. While the GPU is busy crunching numbers, the CPU can work on other tasks, like preparing the next batch of data.

## CUDA Streams

A CUDA Stream is an ordered queue of GPU operations. Operations submitted to a single stream execute in order, one after another. However, operations across different streams can execute concurrently — the GPU can juggle multiple independent workloads at the same time.

## Host-Device Synchronization

Accessing GPU data from the CPU can trigger a Host-Device Synchronization, a common performance bottleneck. This occurs whenever the CPU needs a result from the GPU that isn’t yet available in the CPU’s RAM.

## Why this gets complicated for your platform team:

The device plugin model treats GPUs as opaque, countable integers. When a pod says nvidia.com/gpu: 1, it gets a GPU — but you can't express preferences like "I need two GPUs connected by NVLink" or "give me a GPU with 80GB VRAM." 

There's no topology awareness, no sharing semantics, and no way to do partial allocation (like GPU time-slicing at the scheduler level). 

Each vendor solves these gaps differently with their own custom config (NVIDIA has gpu-feature-discovery, gpu-operator, MIG config, time-slicing config), which means your platform team ends up managing vendor-specific complexity per node type — the opposite of the clean abstraction you have for CPU.

## How DRA (Dynamic Resource Allocation) Fixes This

DRA, which graduated to beta in k8s 1.32, is essentially a rethink of how Kubernetes handles hardware resources. Instead of the "opaque integer counter" model, it introduces a structured, expressive resource model.

DRA is a Kubernetes extension that moves resource management out of the kubelet and into user-space controllers. Instead of the kubelet directly allocating devices, the scheduler makes a request to a DRA controller, which then allocates the resource and configures the node. This lets you build custom allocation logic that understands topology, sharing, and vendor-specific features.

![Device Plugin vs DRA](image.png)

## Key Differences

| Feature | Device Plugin | DRA |
|---------|---------------|-----|
| **Resource Model** | Opaque integer counters | Structured, expressive resources |
| **Allocation** | Kubelet-managed | Controller-managed |
| **Topology Awareness** | None | Built-in |
| **Sharing** | None | Built-in |
| **Vendor-Specific Features** | Custom config per vendor | First-class support |

## Why This Matters for Your Platform Team

DRA solves the biggest pain point for platform teams managing GPUs in Kubernetes: the lack of flexibility and expressiveness in the device plugin model. With DRA, you can:

- Build custom allocation logic that understands topology, sharing, and vendor-specific features
- Express preferences like "I need two GPUs connected by NVLink" or "give me a GPU with 80GB VRAM"
- Avoid vendor-specific complexity by using a standardized API
- Enable advanced features like GPU time-slicing and MIG configuration

## Key API Objects in DRA

- ResourceSlice: Published by driver. Advertises devices + attributes on each node.
- DeviceClass: Cluster-scoped. Defines a category of device (e.g. "gpu") with default selectors.
- ResourceClaimTemplate: Platform-managed blueprint. Teams reference it without knowing device details.
- ResourceClaim: Per-pod request. Specifies what the workload needs via CEL selectors.

## Getting Started with DRA

To get started with DRA, you'll need to:

1. Enable the DRA feature gate in your Kubernetes cluster
2. Install a DRA controller (e.g., NVIDIA DRA controller, AMD DRA controller)
3. Configure your nodes with DRA-specific settings
4. Update your pod specs to use DRA resources

For more information, see the [Kubernetes DRA documentation](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-resources/dynamic-resource-allocation/).

