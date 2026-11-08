# Tesla K80 / Vulkan 1.2 compatibility plan

Status: investigation branch; no K80 execution backend is enabled by this change.

## Why this work exists

The current GPU backend in `src/llm/gpu.c` rejects physical devices whose reported
Vulkan API version is below 1.3. Its device selection also requires buffer device
addresses, shader integer dot product, and subgroup ballot, arithmetic, clustered,
and shuffle operations in compute shaders. The shader build currently targets
Vulkan 1.3. Consequently, a Vulkan 1.2 device is rejected before the engine can
establish which independent operations it can support.

The Tesla K80 is a Kepler GK210 device. Its hardware generation does not provide
the modern integer-dot-product instruction path expected by the existing shader
backend. A Vulkan 1.2 API version alone does not imply that the existing shaders
can run. The compatibility backend must use a separately validated shader set
and capability requirements.

## Change in this branch

`tools/vkcheck.c` now distinguishes a Vulkan 1.2 device from a device below
Vulkan 1.2 before it queries Vulkan 1.3 property and feature structures. It
labels Vulkan 1.2 as a compatibility candidate while explicitly stating that
the current Janas GPU backend is not supported yet.

This is diagnostic-only. It does not change production device selection,
enable a Vulkan 1.2 backend, or claim that a K80 can execute Janas GPU shaders.

## Required implementation stages

1. **Collect a real device report.** Run `vkcheck` on the target host and retain
   `vulkaninfo --summary`, driver version, physical-device properties, extension
   list, queue families, memory heaps/types, and subgroup properties.
2. **Define backend capability profiles.** Keep the current Vulkan 1.3 profile
   unchanged. Define a separate Vulkan 1.2 profile based on queried feature
   structures and extensions. Do not infer support from the GPU marketing name.
3. **Audit runtime API usage.** Inventory every Vulkan 1.3 core function,
   structure, feature, and extension used by `gpu.c`, `gpu_impl.h`, token
   execution, and memory management. Resolve each dependency through Vulkan
   1.2 core or an explicitly enabled extension, or mark the operation
   unsupported.
4. **Audit shader requirements.** Enumerate uses of integer dot product,
   subgroup arithmetic/clustered/shuffle, subgroup-size control, 64-bit
   arithmetic, and other optional GLSL/SPIR-V capabilities. Compile a distinct
   Vulkan 1.2 shader set; do not simply lower `--target-env`.
5. **Implement scalar/subgroup fallbacks.** For each required operation, add a
   portable shader implementation only where it is correct and benchmarkable.
   Preserve CPU fallback for unsupported operations.
6. **Validate numerical behavior.** Compare every GPU kernel against its CPU
   reference over boundary dimensions, quantization formats, non-aligned row
   counts, and randomized inputs. Retain bit-exact guarantees where promised.
7. **Benchmark before enabling by default.** Measure transfer, dispatch,
   synchronization, prompt processing, and token generation separately.
   Enable a K80 path only when it is correct and faster than CPU for the
   relevant workload.

## Important K80 constraints

- Treat the two GK210 GPUs on a K80 board as two independent Vulkan physical
  devices unless the driver explicitly exposes otherwise. Do not assume unified
  memory or automatic pooling of the two 12-GB memory banks.
- Do not assume subgroup width, subgroup operations, buffer device address,
  external host-memory import, or integer-dot-product support. Query each.
- Do not require external host-memory import for a discrete GPU if a device-local
  staging/copy path is implemented.
- A Vulkan API version is not a proxy for shader feature support.
- Keep the CPU implementation as the correctness oracle and reliable fallback.

## Initial acceptance criteria

- `vkcheck` runs without querying Vulkan 1.3 structures on a Vulkan 1.2 device.
- The report clearly separates “Vulkan 1.2 candidate” from “Janas backend
  supported”.
- Existing Vulkan 1.3 devices retain the same acceptance decisions and checks.
- The production engine remains CPU-only on the K80 until the dedicated backend
  passes kernel correctness and performance tests.
