# syntax=docker/dockerfile:1

ARG BASE=ghcr.io/iximiuz/labs/rootfs:ubuntu-docker

# =============================================================================
# Stage 1: toolchain
# Builds Rust into /usr/local so it is shared by root and the lab user.
# Nothing from this stage except the two directories reaches the final image.
# =============================================================================
FROM ${BASE} AS toolchain

ARG RUST_VERSION=stable

USER root
ENV HOME=/root \
    RUSTUP_HOME=/usr/local/rustup \
    CARGO_HOME=/usr/local/cargo \
    PATH=/usr/local/cargo/bin:$PATH

RUN apt-get update \
    && apt-get install -y --no-install-recommends ca-certificates curl \
    && rm -rf /var/lib/apt/lists/*

# minimal profile drops rust-docs (~500 MiB per toolchain); the playground
# links to doc.rust-lang.org instead. rust-src is required by rust-analyzer.
RUN curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs \
    | sh -s -- -y \
        --default-toolchain "${RUST_VERSION}" \
        --profile minimal \
        --no-modify-path \
        -c rustfmt \
        -c clippy \
        -c rust-src \
        -t x86_64-unknown-linux-musl

RUN rustup toolchain install beta --profile minimal -c rustfmt -c clippy \
    && rustup toolchain install nightly --profile minimal -c rustfmt -c clippy

# Drop installer scratch space and make the tree writable here, so the final
# image does not need a recursive chmod layer over ~1 GiB of files.
RUN rm -rf "${RUSTUP_HOME}/downloads" "${RUSTUP_HOME}/tmp" \
           "${CARGO_HOME}/registry" "${CARGO_HOME}/git" \
    && chmod -R a+w "${RUSTUP_HOME}" "${CARGO_HOME}"

# =============================================================================
# Stage 2: cargo tools
# build-essential and the crates.io cache live and die here.
# =============================================================================
FROM toolchain AS cargo-tools

RUN apt-get update \
    && apt-get install -y --no-install-recommends build-essential \
    && rm -rf /var/lib/apt/lists/*

RUN cargo install cargo-watch --locked --root /opt/cargo-tools

# =============================================================================
# Stage 3: final rootfs
# =============================================================================
FROM ${BASE}

ARG LAB_USER=laborant

USER root
ENV HOME=/root \
    RUSTUP_HOME=/usr/local/rustup \
    CARGO_HOME=/usr/local/cargo \
    PATH=/usr/local/cargo/bin:$PATH

# Runtime deps only: a C toolchain and linker for cargo build, pkg-config and
# libssl-dev for the common -sys crates, musl-tools for the static target.
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        build-essential \
        ca-certificates \
        curl \
        git \
        libssl-dev \
        musl-tools \
        pkg-config \
    && rm -rf /var/lib/apt/lists/*

COPY --from=toolchain /usr/local/rustup /usr/local/rustup
COPY --from=toolchain /usr/local/cargo  /usr/local/cargo
COPY --from=cargo-tools --chmod=0755 /opt/cargo-tools/bin/cargo-watch /usr/local/cargo/bin/cargo-watch

# Put every rustup shim on the default PATH.
RUN ln -sf /usr/local/cargo/bin/* /usr/local/bin/

# A rootfs image loses Docker ENV at boot, so the toolchain must also be
# findable with no environment set at all. Default lookup paths are symlinked
# to the shared install, and /etc/profile.d covers login shells.
RUN ln -s "${RUSTUP_HOME}" /root/.rustup \
    && ln -s "${CARGO_HOME}" /root/.cargo \
    && ln -s "${RUSTUP_HOME}" "/home/${LAB_USER}/.rustup" \
    && ln -s "${CARGO_HOME}" "/home/${LAB_USER}/.cargo" \
    && chown -h "${LAB_USER}:" "/home/${LAB_USER}/.rustup" "/home/${LAB_USER}/.cargo"

RUN printf '%s\n' \
        'export RUSTUP_HOME=/usr/local/rustup' \
        'export CARGO_HOME=/usr/local/cargo' \
        'export PATH=$PATH:/usr/local/cargo/bin' \
        > /etc/profile.d/rust.sh \
    && chmod 0644 /etc/profile.d/rust.sh \
    && printf '\n. /etc/profile.d/rust.sh\n' >> /root/.bashrc \
    && printf '\n. /etc/profile.d/rust.sh\n' >> "/home/${LAB_USER}/.bashrc"

USER $LAB_USER
ENV HOME=/home/$LAB_USER

RUN code-server --install-extension rust-lang.rust-analyzer \
    && code-server --install-extension vadimcn.vscode-lldb

COPY --chown=${LAB_USER} welcome /home/${LAB_USER}/.welcome

# Build-time only, adds no layer weight.
RUN rustc --version \
    && cargo --version \
    && rustfmt --version \
    && cargo clippy --version \
    && cargo watch --version \
    && cargo +nightly --version
