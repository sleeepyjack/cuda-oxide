# syntax=docker/dockerfile:1.7
#
# cuda-oxide devcontainer.
#
# Base: NVIDIA CUDA 13.2.1 development image on Ubuntu 24.04 (Noble).
# This is the same upstream image rapidsai/devcontainers builds on top of, but
# without conda/RAPIDS or the C++ build tooling cuda-oxide does not need.
#
# Adds:
#   - LLVM 21 + clang 21 from apt.llvm.org (NVPTX backend; bindgen libclang;
#     clang resource-dir stddef.h that the host `cuda-bindings` build wants).
#   - rustup with the `nightly-2026-04-03` toolchain pinned in
#     rust-toolchain.toml, plus rust-src / rustc-dev / rust-analyzer / clippy.
#
# `cargo oxide doctor` in the running container should report a clean
# environment.

ARG CUDA_IMAGE_TAG=13.2.1-devel-ubuntu24.04
FROM nvidia/cuda:${CUDA_IMAGE_TAG}

ENV DEBIAN_FRONTEND=noninteractive

# ---- Base OS tooling ------------------------------------------------------

RUN apt-get update \
 && apt-get install -y --no-install-recommends \
        build-essential \
        ca-certificates \
        curl \
        git \
        gnupg \
        lsb-release \
        sudo \
 && rm -rf /var/lib/apt/lists/*

# ---- LLVM 21 (NVPTX backend, libclang for bindgen) ------------------------

RUN install -d -m 0755 /etc/apt/keyrings \
 && curl -fsSL https://apt.llvm.org/llvm-snapshot.gpg.key \
        | gpg --dearmor -o /etc/apt/keyrings/llvm.gpg \
 && echo "deb [signed-by=/etc/apt/keyrings/llvm.gpg] http://apt.llvm.org/noble/ llvm-toolchain-noble-21 main" \
        > /etc/apt/sources.list.d/llvm-21.list \
 && apt-get update \
 && apt-get install -y --no-install-recommends \
        llvm-21 \
        llvm-21-dev \
        clang-21 \
        libclang-21-dev \
        libclang-common-21-dev \
 && rm -rf /var/lib/apt/lists/*

# Make the unsuffixed names available without overriding system clang.
RUN update-alternatives --install /usr/bin/llc llc /usr/bin/llc-21 100 \
 && update-alternatives --install /usr/bin/clang clang /usr/bin/clang-21 100

# ---- Non-root user (VS Code devcontainer convention) ---------------------

ARG USERNAME=vscode
ARG USER_UID=1000
ARG USER_GID=${USER_UID}

RUN groupadd --gid ${USER_GID} ${USERNAME} \
 && useradd --uid ${USER_UID} --gid ${USER_GID} -m -s /bin/bash ${USERNAME} \
 && echo "${USERNAME} ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/${USERNAME} \
 && chmod 0440 /etc/sudoers.d/${USERNAME}

USER ${USERNAME}

# ---- Rust toolchain (nightly pinned by rust-toolchain.toml) --------------

ENV RUSTUP_HOME=/home/${USERNAME}/.rustup \
    CARGO_HOME=/home/${USERNAME}/.cargo \
    PATH=/home/${USERNAME}/.cargo/bin:/usr/local/cuda/bin:${PATH}

RUN curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs \
        | sh -s -- -y --default-toolchain none --profile minimal --no-modify-path \
 && rustup toolchain install nightly-2026-04-03 \
        --component rust-src \
        --component rustc-dev \
        --component rust-analyzer \
        --component clippy \
        --component rustfmt \
        --component llvm-tools-preview \
 && rustup default nightly-2026-04-03

# ---- cuda-oxide environment ---------------------------------------------

ENV CUDA_TOOLKIT_PATH=/usr/local/cuda \
    CUDA_OXIDE_LLC=/usr/bin/llc-21 \
    LIBCLANG_PATH=/usr/lib/llvm-21/lib

WORKDIR /workspaces/cuda-oxide
