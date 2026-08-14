# syntax=docker/dockerfile:1

FROM ghcr.io/iximiuz/labs/rootfs:ubuntu-docker

ARG RUST_VERSION=stable
ARG LAB_USER=laborant

USER root

ENV HOME=/root
ENV RUSTUP_HOME=/usr/local/rustup
ENV CARGO_HOME=/usr/local/cargo
ENV PATH=/usr/local/cargo/bin:$PATH

RUN <<EOF
set -eu

apt-get update

apt-get install -y \
  build-essential \
  gcc \
  curl \
  git \
  pkg-config \
  libssl-dev \
  ca-certificates \
  clang \
  lld

# Install Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | \
  sh -s -- -y \
  --default-toolchain ${RUST_VERSION} \
  --profile default \
  --no-modify-path

# Rust tooling
rustup component add rustfmt
rustup component add clippy

# Make Rust available system-wide
ln -sf /usr/local/cargo/bin/rustup /usr/local/bin/rustup
ln -sf /usr/local/cargo/bin/cargo /usr/local/bin/cargo
ln -sf /usr/local/cargo/bin/rustc /usr/local/bin/rustc
ln -sf /usr/local/cargo/bin/rustfmt /usr/local/bin/rustfmt
ln -sf /usr/local/cargo/bin/cargo-fmt /usr/local/bin/cargo-fmt
ln -sf /usr/local/cargo/bin/clippy-driver /usr/local/bin/clippy-driver
ln -sf /usr/local/cargo/bin/cargo-clippy /usr/local/bin/cargo-clippy

printf "\nexport RUSTUP_HOME=/usr/local/rustup\n" >> $HOME/.bashrc
printf "export CARGO_HOME=/usr/local/cargo\n" >> $HOME/.bashrc
printf "export PATH=\$PATH:/usr/local/cargo/bin\n" >> $HOME/.bashrc

rustc --version
cargo --version
rustfmt --version
cargo clippy --version

rm -rf /var/lib/apt/lists/*
EOF

USER $LAB_USER

ENV HOME=/home/$LAB_USER
ENV RUSTUP_HOME=/usr/local/rustup
ENV CARGO_HOME=/usr/local/cargo
ENV PATH=/usr/local/cargo/bin:$PATH

RUN <<EOF
set -eu

printf "\nexport RUSTUP_HOME=/usr/local/rustup\n" >> $HOME/.bashrc
printf "export CARGO_HOME=/usr/local/cargo\n" >> $HOME/.bashrc
printf "export PATH=\$PATH:/usr/local/cargo/bin\n" >> $HOME/.bashrc

rustc --version
cargo --version
rustfmt --version
cargo clippy --version
EOF

# Rust support in VS Code
RUN <<EOF
set -eu

code-server --install-extension rust-lang.rust-analyzer

EOF

COPY welcome $HOME/.welcome
