# syntax=docker/dockerfile:1

FROM ghcr.io/iximiuz/labs/rootfs:ubuntu-docker

ARG RUST_VERSION=stable
ARG LAB_USER=laborant

# ── Environment ───────────────────────────────────────────────────────────────
ENV RUSTUP_HOME=/home/${LAB_USER}/.rustup \
    CARGO_HOME=/home/${LAB_USER}/.cargo \
    PATH=/home/${LAB_USER}/.cargo/bin:$PATH

# ── System dependencies ───────────────────────────────────────────────────────
USER root
ENV HOME=/root

RUN apt-get update && apt-get install -y \
        build-essential \
        ca-certificates \
        clang \
        curl \
        gcc \
        git \
        libssl-dev \
        lld \
        musl-tools \
        pkg-config \
    && rm -rf /var/lib/apt/lists/*

# ── Rust toolchain (installed as lab user — correct ownership from the start) ─
USER $LAB_USER
ENV HOME=/home/$LAB_USER

RUN curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs \
    | sh -s -- -y \
        --default-toolchain "${RUST_VERSION}" \
        --profile default \
        --no-modify-path

RUN rustup toolchain install beta nightly \
    && rustup target add x86_64-unknown-linux-musl \
    && for tc in stable beta nightly; do \
           rustup component add rustfmt clippy --toolchain "$tc"; \
       done

# ── Cargo tools ───────────────────────────────────────────────────────────────
RUN cargo install cargo-watch --locked

# ── System-wide symlinks (needs root) ────────────────────────────────────────
USER root

RUN ln -sf /home/${LAB_USER}/.cargo/bin/rustup          /usr/local/bin/rustup  \
    && ln -sf /home/${LAB_USER}/.cargo/bin/cargo         /usr/local/bin/cargo   \
    && ln -sf /home/${LAB_USER}/.cargo/bin/rustc         /usr/local/bin/rustc   \
    && ln -sf /home/${LAB_USER}/.cargo/bin/rustfmt       /usr/local/bin/rustfmt \
    && ln -sf /home/${LAB_USER}/.cargo/bin/cargo-fmt     /usr/local/bin/cargo-fmt \
    && ln -sf /home/${LAB_USER}/.cargo/bin/clippy-driver /usr/local/bin/clippy-driver \
    && ln -sf /home/${LAB_USER}/.cargo/bin/cargo-clippy  /usr/local/bin/cargo-clippy

# ── Login shell env ───────────────────────────────────────────────────────────
RUN printf 'export RUSTUP_HOME=/home/laborant/.rustup\n'      >  /etc/profile.d/rust.sh \
    && printf 'export CARGO_HOME=/home/laborant/.cargo\n'     >> /etc/profile.d/rust.sh \
    && printf 'export PATH=$PATH:/home/laborant/.cargo/bin\n' >> /etc/profile.d/rust.sh \
    && chmod +x /etc/profile.d/rust.sh

# ── Root shell env ────────────────────────────────────────────────────────────
RUN printf '\nsource /etc/profile.d/rust.sh\n' >> /root/.bashrc

# ── Smoke test ────────────────────────────────────────────────────────────────
USER $LAB_USER
RUN rustc --version \
    && cargo --version \
    && rustfmt --version \
    && cargo clippy --version

# ── Lab user shell env ────────────────────────────────────────────────────────
RUN printf '\nsource /etc/profile.d/rust.sh\n' >> "$HOME/.bashrc"

# ── VS Code extensions ────────────────────────────────────────────────────────
RUN code-server --install-extension rust-lang.rust-analyzer \
    && code-server --install-extension vadimcn.vscode-lldb

# ── Welcome ───────────────────────────────────────────────────────────────────
COPY welcome $HOME/.welcome
