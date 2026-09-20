# syntax=docker/dockerfile:1

ARG BASE=ghcr.io/iximiuz/labs/rootfs:ubuntu-docker

# =============================================================================
# Stage 1: toolchain
# Builds Rust into /usr/local so it is shared by root and the lab user.
# Nothing from this stage except the two directories reaches the final image.
# =============================================================================
FROM ${BASE} AS toolchain

ARG RUST_VERSION=1.98.1

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
# Prebuilt binaries via cargo-binstall; falls back to a source build when a
# crate has no release asset. build-essential and all caches die here.
# =============================================================================
FROM toolchain AS cargo-tools

ARG BINSTALL_VERSION=1.23.0

RUN apt-get update \
    && apt-get install -y --no-install-recommends build-essential \
    && rm -rf /var/lib/apt/lists/*

RUN curl -fsSL \
      "https://github.com/cargo-bins/cargo-binstall/releases/download/v${BINSTALL_VERSION}/cargo-binstall-x86_64-unknown-linux-musl.tgz" \
    | tar -xz -C /usr/local/cargo/bin cargo-binstall

# Cargo workflow tools + Rust-written terminal tools, pinned.
RUN cargo binstall -y --locked --disable-telemetry --root /opt/cargo-tools \
        cargo-watch@8.5.3 \
        bacon@3.25.0 \
        cargo-nextest@0.9.145 \
        cargo-expand@1.0.126 \
        cargo-audit@0.22.2 \
        starship@1.26.0 \
        eza@0.23.5 \
        bat@0.26.1 \
        fd-find@10.5.0 \
        ripgrep@15.2.0 \
        zoxide@0.10.0 \
        git-delta@0.19.2 \
    && cp /usr/local/cargo/bin/cargo-binstall /opt/cargo-tools/bin/

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

# Runtime deps: a C toolchain and linker for cargo build, pkg-config and
# libssl-dev for the common -sys crates, musl-tools for the static target.
# Plus terminal helpers that are not Rust crates.
RUN apt-get update \
    && apt-get upgrade -y \
    && apt-get install -y --no-install-recommends \
        bash-completion \
        btop \
        build-essential \
        ca-certificates \
        curl \
        fzf \
        git \
        jq \
        less \
        libssl-dev \
        musl-tools \
        pkg-config \
        tmux \
        tree \
    && rm -rf /var/lib/apt/lists/*

COPY --from=toolchain /usr/local/rustup /usr/local/rustup
COPY --from=toolchain /usr/local/cargo  /usr/local/cargo
COPY --from=cargo-tools --chmod=0755 /opt/cargo-tools/bin/ /usr/local/bin/

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
    && chmod 0644 /etc/profile.d/rust.sh

# Shell completions for rustup and cargo.
RUN mkdir -p /etc/bash_completion.d \
    && rustup completions bash > /etc/bash_completion.d/rustup \
    && rustup completions bash cargo > /etc/bash_completion.d/cargo

# ─── Prompt theme (no Nerd Font needed, renders in the browser terminal) ──────
RUN <<EOF
set -eu
cat > /etc/starship.toml <<'TOML'
add_newline = true
command_timeout = 1000

format = """
$directory\
$git_branch\
$git_status\
$package\
$rust\
$cmd_duration\
$jobs\
$line_break\
$character"""

[character]
success_symbol = "[❯](bold green)"
error_symbol = "[❯](bold red)"

[directory]
style = "bold cyan"
truncation_length = 3
truncate_to_repo = true

[git_branch]
symbol = "git:"
style = "bold purple"
format = "[$symbol$branch]($style) "

[git_status]
style = "red"
format = "([\\[$all_status$ahead_behind\\]]($style) )"

[package]
symbol = "pkg:"
style = "bold 208"
format = "[$symbol$version]($style) "

[rust]
symbol = "rust:"
style = "bold red"
format = "[$symbol$version]($style) "

[cmd_duration]
min_time = 2000
style = "dimmed yellow"
format = "[took $duration]($style) "

[jobs]
symbol = "bg:"
style = "bold blue"
TOML

cat > /etc/tmux.conf <<'TMUX'
set -g default-terminal "tmux-256color"
set -ga terminal-overrides ",xterm-256color:RGB"
set -g mouse on
set -g history-limit 50000
set -g base-index 1
setw -g pane-base-index 1
set -g status-style "bg=colour236,fg=colour250"
set -g status-left "#[bold,fg=colour208] #S "
set -g status-right "#[fg=colour245]%H:%M "
TMUX

git config --system core.pager delta
git config --system interactive.diffFilter "delta --color-only"
git config --system delta.navigate true
git config --system delta.line-numbers true
git config --system merge.conflictStyle zdiff3
EOF

# ─── Interactive shell, shared by root and $LAB_USER ──────────────────────────
RUN <<EOF
set -eu
cat > /etc/profile.d/rust-shell.sh <<'PROFILE'
export STARSHIP_CONFIG=/etc/starship.toml
export CARGO_TERM_COLOR=always
export RUST_BACKTRACE=1
export BAT_THEME=TwoDark
export MANPAGER="sh -c 'col -bx | bat -l man -p'"
export FZF_DEFAULT_COMMAND='fd --type f --hidden --exclude .git --exclude target'
export FZF_DEFAULT_OPTS='--height 40% --layout=reverse --border'

# Everything below is interactive bash only, and runs once per shell.
[ -n "${BASH_VERSION:-}" ] || return 0
case $- in *i*) ;; *) return 0 ;; esac
[ -z "${__RUST_BASH_INIT:-}" ] || return 0
__RUST_BASH_INIT=1

HISTSIZE=10000
HISTFILESIZE=20000
HISTCONTROL=ignoreboth:erasedups
shopt -s histappend

alias c=cargo
alias cb='cargo build'
alias cr='cargo run'
alias ct='cargo test'
alias cn='cargo nextest run'
alias cl='cargo clippy'
alias cf='cargo fmt'

alias ls='eza --group-directories-first'
alias ll='eza -l --git --group-directories-first'
alias la='eza -la --git --group-directories-first'
alias lt='eza --tree --level=2 --group-directories-first --git-ignore'

[ -f /usr/share/doc/fzf/examples/key-bindings.bash ] && . /usr/share/doc/fzf/examples/key-bindings.bash
[ -f /usr/share/bash-completion/completions/fzf ] && . /usr/share/bash-completion/completions/fzf

eval "$(zoxide init bash)"
eval "$(starship init bash)"
PROFILE
chmod 0644 /etc/profile.d/rust-shell.sh
for rc in /root/.bashrc "/home/${LAB_USER}/.bashrc"; do
  printf '\n. /etc/profile.d/rust.sh\n. /etc/profile.d/rust-shell.sh\n' >> "$rc"
done
EOF

USER $LAB_USER
ENV HOME=/home/$LAB_USER

RUN code-server --install-extension rust-lang.rust-analyzer \
    && code-server --install-extension vadimcn.vscode-lldb \
    && code-server --install-extension tamasfe.even-better-toml

# code-server: login bash terminal (loads the prompt), clippy on save,
# inlay hints. Merges into any settings the base image already ships.
RUN <<EOF
set -eu
dir="$HOME/.local/share/code-server/User"
mkdir -p "$dir"
cat > /tmp/rust-settings.json <<'JSON'
{
  "workbench.colorTheme": "Default Dark Modern",
  "terminal.integrated.defaultProfile.linux": "bash",
  "terminal.integrated.profiles.linux": {
    "bash": { "path": "bash", "args": ["-l"] }
  },
  "terminal.integrated.fontSize": 14,
  "terminal.integrated.lineHeight": 1.2,
  "terminal.integrated.cursorStyle": "line",
  "terminal.integrated.cursorBlinking": true,
  "terminal.integrated.smoothScrolling": true,
  "terminal.integrated.scrollback": 10000,
  "editor.fontSize": 14,
  "editor.smoothScrolling": true,
  "editor.cursorSmoothCaretAnimation": "on",
  "editor.bracketPairColorization.enabled": true,
  "editor.guides.bracketPairs": "active",
  "editor.inlayHints.enabled": "onUnlessPressed",
  "files.watcherExclude": { "**/target/**": true },
  "rust-analyzer.check.command": "clippy",
  "rust-analyzer.inlayHints.chainingHints.enable": true,
  "rust-analyzer.inlayHints.parameterHints.enable": true,
  "[rust]": {
    "editor.defaultFormatter": "rust-lang.rust-analyzer",
    "editor.formatOnSave": true
  },
  "[toml]": {
    "editor.defaultFormatter": "tamasfe.even-better-toml"
  }
}
JSON
if [ -s "$dir/settings.json" ]; then
  jq -s '.[0] * .[1]' "$dir/settings.json" /tmp/rust-settings.json > /tmp/rust-merged.json
  mv /tmp/rust-merged.json "$dir/settings.json"
else
  mv /tmp/rust-settings.json "$dir/settings.json"
fi
rm -f /tmp/rust-settings.json
EOF

COPY --chown=${LAB_USER} welcome /home/${LAB_USER}/.welcome

# Build-time only, adds no layer weight.
RUN rustc --version \
    && cargo --version \
    && rustfmt --version \
    && cargo clippy --version \
    && cargo +nightly --version \
    && cargo watch --version \
    && bacon --version \
    && cargo nextest --version \
    && cargo expand --version \
    && cargo audit --version \
    && cargo binstall -V \
    && starship --version \
    && eza --version \
    && bat --version \
    && fd --version \
    && rg --version \
    && zoxide --version \
    && delta --version
