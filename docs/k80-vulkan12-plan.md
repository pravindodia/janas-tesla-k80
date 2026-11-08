# Tesla K80 / Vulkan 1.2 compatibility

Status: experimental restricted runtime path added; not yet hardware-validated.

## Supported scope

The Vulkan 1.2 path is deliberately limited to **Q4_K matrix-vector products with
one activation vector**. It uses the separate
`src/llm/shaders/vulkan12/q4k_matvec.comp` shader, which replaces the integer
dot-product instruction with explicit signed-byte unpacking and scalar integer
multiplication. The build generates a Vulkan 1.2 SPIR-V C array and validates
the SPIR-V when `spirv-val` is installed.

The device-selection path queries only Vulkan 1.1 and 1.2 structures for a
Vulkan 1.2 physical device. It requires buffer device address, compute queue,
the subgroup operations checked by the backend, and a fixed subgroup size of
32. The Vulkan 1.3 path remains separate and retains its existing pipeline
initialization.

For Vulkan 1.2 devices, Janas creates only the Q4_K single-vector pipeline.
Other quantized formats, multi-vector operations, expert execution, dense
matmul, and whole-token GPU execution remain on the CPU. Startup self-check
compares Q4_K results bit-for-bit with the CPU; a mismatch disables Q4_K GPU
dispatch for that device/driver combination.

## Validation on the Tesla K80

Run on the target host:

```bash
CC=gcc-13 HOSTCC=gcc-13 ./build.sh release clean-build
./bin/x86_64-linux/vkcheck
./bin/x86_64-linux/test_quant
JANAS_GPU_SELFCHECK=2 JANAS_GPU_VERBOSE=1 ./bin/x86_64-linux/gpu_check
```

The `gpu_check` run must show the K80 selected and Q4_K results matching the
CPU. If the self-check rejects Q4_K, GPU dispatch remains disabled and the
CPU path continues to work. Run a small Q4_K model only after the numerical
check passes. Record the two K80 Vulkan physical devices separately and
benchmark token generation against CPU-only execution before widening scope.

This is experimental. A successful compile or device probe is not proof of
working inference. Keep the CPU implementation as the fallback.
