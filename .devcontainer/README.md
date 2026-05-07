# cuda-oxide devcontainer

Minimal devcontainer that boots a GPU-attached cuda-oxide build environment.

## What's inside

- `nvidia/cuda:13.2.1-devel-ubuntu24.04` (CUDA Toolkit + driver stub +
  `cuda-gdb`)
- LLVM 21 + clang 21 from `apt.llvm.org` (NVPTX backend, libclang for the
  `cuda-bindings` build, `stddef.h` resource-dir headers)
- rustup with `nightly-2026-04-03` (the channel `rust-toolchain.toml` pins),
  components: `rust-src`, `rustc-dev`, `rust-analyzer`, `clippy`, `rustfmt`,
  `llvm-tools-preview`
- non-root `vscode` user with passwordless sudo

## Why not the rapidsai prebuilt images

cuda-oxide is a pure Rust workspace; it does not need conda, cmake, ninja,
sccache, or the RAPIDS C++ stack that ships in `rapidsai/devcontainers:*`.
Their published image tags also pin LLVM below 21, and cuda-oxide requires
LLVM 21+ to lower TMA / tcgen05 / WGMMA intrinsics. The base image used here
is the same upstream `nvidia/cuda` devel image rapidsai builds on top of, just
without the extra layers cuda-oxide does not exercise.

## Host requirements

- A working install of the
  [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/)
  on the host so the container can see the GPU.
- NVIDIA driver **580 or newer** (CUDA 13.2 toolkit minimum). Older drivers
  produce "Driver/library version mismatch" inside the container even though
  the host's `nvidia-smi` works.
- An Ampere-or-newer (sm_80+) GPU. Hopper / Blackwell features depend on the
  matching architecture being present at runtime (compile-time is fine
  regardless because PTX generation is GPU-agnostic).

## Usage

From VS Code: open the repository, choose **Dev Containers: Reopen in
Container**.

From the CLI with the `devcontainer` tool:

```bash
devcontainer up --workspace-folder .
devcontainer exec --workspace-folder . cargo oxide run vecadd
```

`postCreateCommand` chains `cargo oxide setup` (compiles the
`librustc_codegen_cuda.so` backend once during container creation, so the
first interactive `cargo oxide run` is fast) and `cargo oxide doctor` (so
you see the prerequisite matrix immediately on first start). The combined
command is suffixed with `|| true` so an incomplete environment does not
abort the container build.

The codegen backend artifacts land in the bind-mounted workspace `target/`,
so they survive image rebuilds without re-baking the image.

## Quick verification once attached

```bash
nvidia-smi                    # GPU is visible
cargo oxide doctor            # all green
cargo oxide run vecadd        # builds + runs the canonical example
cargo oxide pipeline vecadd   # dump every IR stage end-to-end
```

## Customising the CUDA version

The base image tag is wired through a build arg in `devcontainer.json` under
`build.args.CUDA_IMAGE_TAG`. Edit it there to pin a different
`nvidia/cuda:*-devel-ubuntu24.04` tag, then rebuild the container.

## Optional: libmathdx

`.cargo/config.toml` ships a `LIBMATHDX_PATH = "/path/to/libmathdx/install"`
placeholder. It is only consulted by the `mathdx_ffi_test` example, which is
opt-in (`cargo oxide run mathdx_ffi_test --features libmathdx`). Leave the
placeholder alone or override it via env (`LIBMATHDX_PATH=/your/path
cargo oxide ...`) at run time. Every other example builds without
libmathdx.
