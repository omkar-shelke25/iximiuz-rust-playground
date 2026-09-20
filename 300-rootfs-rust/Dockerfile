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
# Stage 2: tools
# Prebuilt binaries via cargo-binstall (falls back to a source build when a
# crate has no release asset), plus lazygit and fastfetch release archives.
# build-essential and all caches die here.
# =============================================================================
FROM toolchain AS tools

ARG BINSTALL_VERSION=1.23.0
ARG LAZYGIT_VERSION=0.65.1
ARG FASTFETCH_VERSION=2.68.1

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
        atuin@18.22.0 \
    && cp /usr/local/cargo/bin/cargo-binstall /opt/cargo-tools/bin/

RUN curl -fsSL \
      "https://github.com/jesseduffield/lazygit/releases/download/v${LAZYGIT_VERSION}/lazygit_${LAZYGIT_VERSION}_linux_x86_64.tar.gz" \
    | tar -xz -C /opt/cargo-tools/bin lazygit

RUN mkdir -p /opt/fastfetch \
    && curl -fsSL \
      "https://github.com/fastfetch-cli/fastfetch/releases/download/${FASTFETCH_VERSION}/fastfetch-linux-amd64.tar.gz" \
    | tar -xz -C /opt/fastfetch --strip-components=2

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
# Plus zsh and its plugins. The base image already ships fzf (~/.fzf),
# btop, jq and yq.
RUN apt-get update \
    && apt-get upgrade -y \
    && apt-get install -y --no-install-recommends \
        bash-completion \
        build-essential \
        ca-certificates \
        curl \
        git \
        less \
        libssl-dev \
        musl-tools \
        pkg-config \
        tmux \
        tree \
        zsh \
        zsh-autosuggestions \
        zsh-syntax-highlighting \
    && rm -rf /var/lib/apt/lists/*

COPY --from=toolchain /usr/local/rustup /usr/local/rustup
COPY --from=toolchain /usr/local/cargo  /usr/local/cargo
COPY --from=tools --chmod=0755 /opt/cargo-tools/bin/ /usr/local/bin/
COPY --from=tools /opt/fastfetch/ /usr/local/

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

# Shell completions for rustup and cargo, bash and zsh.
RUN mkdir -p /etc/bash_completion.d /usr/local/share/zsh/site-functions \
    && rustup completions bash > /etc/bash_completion.d/rustup \
    && rustup completions bash cargo > /etc/bash_completion.d/cargo \
    && rustup completions zsh > /usr/local/share/zsh/site-functions/_rustup \
    && rustup completions zsh cargo > /usr/local/share/zsh/site-functions/_cargo

# ─── Prompt themes: Catppuccin Mocha ──────────────────────────────────────────
# starship.toml uses Nerd Font glyphs and powerline arrows (`icons on`).
# starship-plain.toml uses colored blocks only, the default for browser tabs.
RUN <<EOF
set -eu
cat > /etc/starship.toml <<'TOML'
"$schema" = 'https://starship.rs/config-schema.json'

format = """
[\ue0b6](red)\
$os\
$hostname\
[\ue0b0](bg:peach fg:red)\
${env_var.PWD}\
[\ue0b0](bg:yellow fg:peach)\
$git_branch\
$git_status\
[\ue0b0](fg:yellow bg:green)\
$rust\
$package\
[\ue0b0](fg:green bg:lavender)\
$time\
[\ue0b4 ](fg:lavender)\
$cmd_duration\
$jobs\
$line_break\
$status\
$character"""

palette = 'catppuccin_mocha'

[palettes.catppuccin_mocha]
rosewater = "#f5e0dc"
flamingo = "#f2cdcd"
pink = "#f5c2e7"
mauve = "#cba6f7"
red = "#f38ba8"
maroon = "#eba0ac"
peach = "#fab387"
yellow = "#f9e2af"
green = "#a6e3a1"
teal = "#94e2d5"
sky = "#89dceb"
sapphire = "#74c7ec"
blue = "#89b4fa"
lavender = "#b4befe"
text = "#cdd6f4"
subtext1 = "#bac2de"
subtext0 = "#a6adc8"
overlay2 = "#9399b2"
overlay1 = "#7f849c"
overlay0 = "#6c7086"
surface2 = "#585b70"
surface1 = "#45475a"
surface0 = "#313244"
base = "#1e1e2e"
mantle = "#181825"
crust = "#11111b"

[os]
disabled = false
style = "bg:red fg:crust"
format = "[$symbol ]($style)"

[os.symbols]
Ubuntu = "\uf31b"
Linux = "\U000f033d"

[hostname]
ssh_only = false
style = "bg:red fg:crust"
format = "[$hostname ]($style)"

[env_var.PWD]
variable = "PWD"
style = "bg:peach fg:crust"
format = "[ \uf07c $env_value ]($style)"

[git_branch]
symbol = "\uf418"
style = "bg:yellow"
format = "[[ $symbol $branch ](fg:crust bg:yellow)]($style)"

[git_status]
style = "bg:yellow"
format = "[[($all_status$ahead_behind )](fg:crust bg:yellow)]($style)"

[rust]
symbol = "\ue7a8"
style = "bg:green"
format = "[[ $symbol $version ](fg:crust bg:green)]($style)"

[package]
symbol = "\uf487"
style = "bg:green"
format = "[[ $symbol $version ](fg:crust bg:green)]($style)"

[time]
disabled = false
time_format = "%H:%M"
style = "bg:lavender"
format = "[[ \uf017 $time ](fg:crust bg:lavender)]($style)"

[cmd_duration]
min_time = 2000
style = "fg:yellow"
format = "[\uf252 $duration ]($style)"

[jobs]
symbol = "\uf013 "
style = "fg:blue"

[status]
disabled = false
style = "bold fg:red"
format = "[\u2718 $status ]($style)"

[character]
success_symbol = "[\u276f](bold fg:green)"
error_symbol = "[\u276f](bold fg:red)"
TOML

cat > /etc/starship-plain.toml <<'TOML'
add_newline = true
format = "$hostname${env_var.PWD}$git_branch$git_status$rust$package$time$cmd_duration$jobs$line_break$status$character"

[hostname]
ssh_only = false
style = "bg:#f38ba8 fg:#11111b"
format = "[ $hostname ]($style)"

[env_var.PWD]
variable = "PWD"
style = "bg:#fab387 fg:#11111b"
format = "[ $env_value ]($style)"

[git_branch]
symbol = "git:"
style = "bg:#f9e2af fg:#11111b"
format = "[ $symbol$branch ]($style)"

[git_status]
style = "bg:#f9e2af fg:#11111b"
format = "([$all_status$ahead_behind ]($style))"

[rust]
symbol = "rust:"
style = "bg:#a6e3a1 fg:#11111b"
format = "[ $symbol$version ]($style)"

[package]
symbol = "pkg:"
style = "bg:#a6e3a1 fg:#11111b"
format = "[ $symbol$version ]($style)"

[time]
disabled = false
time_format = "%H:%M"
style = "bg:#b4befe fg:#11111b"
format = "[ $time ]($style)"

[cmd_duration]
min_time = 2000
style = "fg:#f9e2af"
format = " [took $duration]($style)"

[jobs]
symbol = "bg:"
style = "fg:#89b4fa"

[status]
disabled = false
style = "bold fg:#f38ba8"
format = "[$status ]($style)"

[character]
success_symbol = "[>](bold fg:#a6e3a1)"
error_symbol = "[>](bold fg:#f38ba8)"
TOML

mkdir -p /etc/fastfetch
cat > /etc/fastfetch/config.jsonc <<'JSON'
{
  "$schema": "https://github.com/fastfetch-cli/fastfetch/raw/dev/doc/json_schema.json",
  "logo": { "source": "ubuntu_small", "padding": { "top": 1, "right": 3 } },
  "display": { "separator": "  " },
  "modules": [
    "title", "separator", "os", "kernel", "uptime", "shell",
    "cpu", "memory", "disk", "localip",
    "break",
    { "type": "command", "key": "Rust", "text": "rustc --version | cut -d' ' -f2" },
    { "type": "command", "key": "Cargo", "text": "cargo --version | cut -d' ' -f2" },
    { "type": "command", "key": "Docker", "text": "docker --version | cut -d' ' -f3 | tr -d ," },
    "break", "colors"
  ]
}
JSON

cat > /etc/tmux.conf <<'TMUX'
set -g default-terminal "tmux-256color"
set -ga terminal-overrides ",xterm-256color:RGB"
set -g mouse on
set -g history-limit 50000
set -g base-index 1
setw -g pane-base-index 1
set -g status-style "bg=#313244,fg=#cdd6f4"
set -g status-left "#[bold,fg=#11111b,bg=#fab387] #S "
set -g status-right "#[fg=#11111b,bg=#b4befe] %H:%M "
TMUX

git config --system core.pager delta
git config --system interactive.diffFilter "delta --color-only"
git config --system delta.navigate true
git config --system delta.line-numbers true
git config --system delta.syntax-theme "Catppuccin Mocha"
git config --system merge.conflictStyle zdiff3
EOF

# ─── Shared env, sourced by bash and zsh ──────────────────────────────────────
RUN <<EOF
set -eu
cat > /etc/profile.d/rust-shell.sh <<'PROFILE'
export CARGO_TERM_COLOR=always
export RUST_BACKTRACE=1
export BAT_THEME="Catppuccin Mocha"
export MANPAGER="sh -c 'col -bx | bat -l man -p'"

# Bash fallback: plain prompt and the basics. zsh is the default shell.
[ -n "${BASH_VERSION:-}" ] || return 0
case $- in *i*) ;; *) return 0 ;; esac
[ -z "${__RUST_BASH_INIT:-}" ] || return 0
__RUST_BASH_INIT=1

export STARSHIP_CONFIG=/etc/starship-plain.toml
alias c=cargo cb='cargo build' cr='cargo run' ct='cargo test'
alias cn='cargo nextest run' cl='cargo clippy' cf='cargo fmt'
alias ls='eza --group-directories-first'
alias ll='eza -la --git --group-directories-first'
eval "$(zoxide init bash)"
eval "$(starship init bash)"
PROFILE
chmod 0644 /etc/profile.d/rust-shell.sh
for rc in /root/.bashrc "/home/${LAB_USER}/.bashrc"; do
  printf '\n. /etc/profile.d/rust.sh\n. /etc/profile.d/rust-shell.sh\n' >> "$rc"
done
EOF

# ─── zsh: system-wide config, per-user stubs, default shell ───────────────────
RUN <<EOF
set -eu
cat > /etc/zsh/playground.zsh <<'ZSH'
# Shared zsh config for the Rust playground.
# Personal additions go in ~/.zshrc.local

emulate sh -c '. /etc/profile.d/rust.sh; . /etc/profile.d/rust-shell.sh'
typeset -U path PATH
export LANG=${LANG:-C.UTF-8}
export EDITOR=${EDITOR:-vim} VISUAL=${VISUAL:-vim}

# History
HISTFILE=~/.zsh_history
HISTSIZE=100000
SAVEHIST=100000
setopt SHARE_HISTORY HIST_IGNORE_ALL_DUPS HIST_IGNORE_SPACE HIST_REDUCE_BLANKS INTERACTIVE_COMMENTS

# Keys
bindkey -e
bindkey '^[[H' beginning-of-line
bindkey '^[[F' end-of-line
bindkey '^[[3~' delete-char
bindkey '^[[1;5C' forward-word
bindkey '^[[1;5D' backward-word

# Completion (_rustup and _cargo live in /usr/local/share/zsh/site-functions)
autoload -Uz compinit && compinit -i
zstyle ':completion:*' menu select
zstyle ':completion:*' matcher-list 'm:{a-z}={A-Z}'

# Aliases
alias cat='bat --paging=never --style=plain'
alias grep='grep --color=auto'
alias top='btop'
alias lg='lazygit'
alias c='cargo'
alias cb='cargo build'
alias cr='cargo run'
alias ct='cargo test'
alias cn='cargo nextest run'
alias cl='cargo clippy'
alias cf='cargo fmt'

# fzf: Ctrl+T files, Alt+C directories (Ctrl+R is atuin)
export FZF_DEFAULT_COMMAND='fd --type f --hidden --exclude .git --exclude target'
export FZF_CTRL_T_COMMAND=$FZF_DEFAULT_COMMAND
export FZF_ALT_C_COMMAND='fd --type d --hidden --exclude .git --exclude target'
export FZF_DEFAULT_OPTS='--height 50% --layout=reverse --border=rounded --info=inline --color=bg+:#313244,spinner:#f5e0dc,hl:#f38ba8,fg:#cdd6f4,header:#f38ba8,info:#cba6f7,pointer:#f5e0dc,marker:#b4befe,fg+:#cdd6f4,prompt:#cba6f7,hl+:#f38ba8,selected-bg:#45475a,border:#6c7086,label:#cdd6f4'
export FZF_CTRL_T_OPTS="--preview 'bat --color=always --style=numbers --line-range=:200 {}'"
if [[ -f ~/.fzf.zsh ]]; then
  source ~/.fzf.zsh
elif (( $+commands[fzf] )); then
  source <(fzf --zsh)
fi

# Icons need a Nerd Font on the machine that draws the terminal.
# Browser tabs cannot load one, so icons default to off. They turn on for
# Ghostty, Kitty and WezTerm over `labctl ssh`. `icons on|off` overrides
# and remembers the choice.
_icons_default() {
  if [[ -f $HOME/.config/term-icons ]]; then print -r -- "$(<$HOME/.config/term-icons)"; return; fi
  case "$TERM" in
    xterm-ghostty|xterm-kitty|wezterm) print on ;;
    *) print off ;;
  esac
}
_apply_icons() {
  local i=--icons=auto
  export STARSHIP_CONFIG=/etc/starship.toml
  if [[ $TERM_ICONS == off ]]; then
    i=--icons=never
    export STARSHIP_CONFIG=/etc/starship-plain.toml
  fi
  alias ls="eza $i --group-directories-first"
  alias ll="eza -la $i --git --group-directories-first"
  alias tree="eza --tree $i --git-ignore"
  export FZF_ALT_C_OPTS="--preview 'eza --tree --level=2 ${i/auto/always} --color=always {}'"
}
icons() {
  [[ $1 == on || $1 == off ]] || { print "usage: icons on|off"; return 1; }
  TERM_ICONS=$1
  mkdir -p $HOME/.config
  print -r -- $1 > $HOME/.config/term-icons
  _apply_icons
}
TERM_ICONS=$(_icons_default)
_apply_icons

# History search, smart cd, prompt
eval "$(atuin init zsh --disable-up-arrow)"
eval "$(zoxide init zsh)"
eval "$(starship init zsh)"

# System info once per session, then the playground welcome on first login
if [[ -t 1 && -z $FASTFETCH_SHOWN ]]; then
  export FASTFETCH_SHOWN=1
  fastfetch -c /etc/fastfetch/config.jsonc
fi
if [[ -t 0 && -f $HOME/.welcome ]]; then
  if [[ -x $HOME/.welcome ]]; then "$HOME/.welcome"; else command cat "$HOME/.welcome"; fi
  echo
  rm -f "$HOME/.welcome"
fi
ZSH

cat > /etc/zsh/playground-plugins.zsh <<'ZSH'
# Loaded last: syntax highlighting must come after every other widget.
source /usr/share/zsh-autosuggestions/zsh-autosuggestions.zsh
ZSH_AUTOSUGGEST_HIGHLIGHT_STYLE='fg=#6c7086'
ZSH_AUTOSUGGEST_STRATEGY=(history completion)
source /usr/share/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh
ZSH

for home in /root "/home/${LAB_USER}"; do
  # Ubuntu's /etc/zsh/zshrc runs its own compinit; skip it, ours runs once.
  printf 'skip_global_compinit=1\n' > "$home/.zshenv"
  cat > "$home/.zshrc" <<'ZSH'
# Playground defaults live in /etc/zsh. Personal additions go in ~/.zshrc.local
source /etc/zsh/playground.zsh
[[ -f ~/.zshrc.local ]] && source ~/.zshrc.local
source /etc/zsh/playground-plugins.zsh
ZSH
  mkdir -p "$home/.config/atuin"
  cat > "$home/.config/atuin/config.toml" <<'TOML'
update_check = false
style = "compact"
inline_height = 20
show_preview = true
TOML
done
chown -R "${LAB_USER}:" "/home/${LAB_USER}/.zshenv" "/home/${LAB_USER}/.zshrc" "/home/${LAB_USER}/.config"

usermod -s /usr/bin/zsh root
usermod -s /usr/bin/zsh "${LAB_USER}"

zsh -ic 'exit'
rm -f /root/.zcompdump*
EOF

USER $LAB_USER
ENV HOME=/home/$LAB_USER

RUN code-server --install-extension rust-lang.rust-analyzer \
    && code-server --install-extension vadimcn.vscode-lldb \
    && code-server --install-extension tamasfe.even-better-toml \
    && code-server --install-extension catppuccin.catppuccin-vsc

# code-server: zsh login terminal, Catppuccin Mocha theme, clippy on save,
# inlay hints. The font list lets a locally installed Nerd Font draw icons
# after `icons on`. Merges into the settings the base image already ships.
RUN <<EOF
set -eu
dir="$HOME/.local/share/code-server/User"
mkdir -p "$dir"
cat > /tmp/rust-settings.json <<'JSON'
{
  "workbench.colorTheme": "Catppuccin Mocha",
  "terminal.integrated.defaultProfile.linux": "zsh",
  "terminal.integrated.profiles.linux": {
    "zsh": { "path": "zsh", "args": ["-l"] },
    "bash": { "path": "bash", "args": ["-l"] }
  },
  "terminal.integrated.fontFamily": "'JetBrainsMono Nerd Font', 'JetBrains Mono', Menlo, Consolas, monospace",
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
    && delta --version \
    && atuin --version \
    && lazygit --version \
    && fastfetch --version \
    && zsh -ic 'exit' \
    && rm -f "$HOME"/.zcompdump*

COPY --chown=${LAB_USER} welcome /home/${LAB_USER}/.welcome
