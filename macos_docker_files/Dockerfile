# Claude Code development container — macOS (Apple Silicon or Intel) host
#
# Build:  docker compose build
# Run:    docker compose run --rm claude
#
# No --platform is set anywhere: on an Apple Silicon MacBook this builds and
# runs native linux/arm64. Do NOT force linux/amd64 unless a tool in your
# project genuinely has no arm64 build — that path goes through Rosetta/QEMU
# emulation and is several times slower.

ARG NODE_VERSION=22
FROM node:${NODE_VERSION}-bookworm

# Pin the CLI for reproducible rebuilds; "latest" tracks the current release.
ARG CLAUDE_CODE_VERSION=latest

# Match these to your Mac account if you ever run this on a Linux host too.
# On Docker Desktop for Mac the UID mismatch that bites on Linux does not
# apply — the VirtioFS layer maps ownership for you — so 1000:1000 is fine.
ARG USERNAME=node
ARG USER_UID=1000
ARG USER_GID=1000

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates \
      curl \
      wget \
      git \
      gnupg \
      openssh-client \
      less \
      jq \
      ripgrep \
      fd-find \
      unzip \
      zip \
      procps \
      man-db \
      sudo \
      zsh \
      vim \
      build-essential \
      python3 \
      python3-pip \
      python3-venv \
    && rm -rf /var/lib/apt/lists/*

# GitHub CLI (multi-arch; used instead of mounting host SSH keys)
RUN install -m 0755 -d /etc/apt/keyrings \
    && curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg \
         -o /etc/apt/keyrings/githubcli-archive-keyring.gpg \
    && chmod go+r /etc/apt/keyrings/githubcli-archive-keyring.gpg \
    && echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" \
         > /etc/apt/sources.list.d/github-cli.list \
    && apt-get update && apt-get install -y --no-install-recommends gh \
    && rm -rf /var/lib/apt/lists/*

# Passwordless sudo for the dev user. Drop this block if you want the
# container to have no escalation path at all.
RUN echo "${USERNAME} ALL=(root) NOPASSWD:ALL" > /etc/sudoers.d/${USERNAME} \
    && chmod 0440 /etc/sudoers.d/${USERNAME}

# Give npm a global prefix the non-root user owns, so `npm i -g` and Claude
# Code's own auto-updater work without sudo.
ENV NPM_CONFIG_PREFIX=/usr/local/share/npm-global
ENV PATH=${NPM_CONFIG_PREFIX}/bin:$PATH
RUN mkdir -p ${NPM_CONFIG_PREFIX} && chown -R ${USER_UID}:${USER_GID} ${NPM_CONFIG_PREFIX}

# ---------------------------------------------------------------------------
# THIS BLOCK IS THE ONE THAT PREVENTS THE "authentication keeps failing"
# PROBLEM. Docker seeds a fresh named volume from whatever is at the mount
# point in the image, INCLUDING its ownership. If /home/node/.claude does not
# exist in the image, Docker creates the volume root-owned, the `node` user
# cannot write .credentials.json, and every login silently fails to persist.
# Creating the directories here, owned by the dev user, seeds the volume
# correctly the first time it is created.
# ---------------------------------------------------------------------------
RUN mkdir -p /home/${USERNAME}/.claude \
             /home/${USERNAME}/.claude/projects \
             /home/${USERNAME}/.cache \
             /home/${USERNAME}/workspace \
    && chown -R ${USER_UID}:${USER_GID} /home/${USERNAME}

USER ${USERNAME}

# Claude Code keeps auth token, settings, session history AND .claude.json
# (OAuth account, MCP servers, per-project trust) under one directory when
# CLAUDE_CONFIG_DIR points at it. Without this, .claude.json lands outside the
# volume and you get signed out on every rebuild.
ENV CLAUDE_CONFIG_DIR=/home/${USERNAME}/.claude
ENV SHELL=/bin/zsh
ENV EDITOR=vim

RUN npm install -g @anthropic-ai/claude-code@${CLAUDE_CODE_VERSION}

WORKDIR /home/${USERNAME}/workspace

CMD ["/bin/zsh"]
