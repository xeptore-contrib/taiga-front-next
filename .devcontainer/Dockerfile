# syntax=docker/dockerfile:1
FROM --platform=$BUILDPLATFORM docker.io/library/ubuntu:24.04 AS git-build

ARG TARGETARCH
ARG BUILDARCH

RUN <<EOT bash
  set -Eexuo pipefail

  case "${BUILDARCH}" in
    amd64|arm64) ;;
    *) echo "Only amd64 or arm64 build architecture is supported" >&2; exit 16 ;;
  esac;

  case "${TARGETARCH}" in
    arm64|amd64) ;;
    *) echo "Only arm64 or amd64 target architecture is supported" >&2; exit 16 ;;
  esac;
EOT

ENV DEBIAN_FRONTEND=noninteractive

COPY ubuntu.sources /etc/apt/sources.list.d/ubuntu.sources

RUN <<EOT bash
  set -Eexuo pipefail

  if [ "$BUILDARCH" = "amd64" ] && [ "$TARGETARCH" = "arm64" ]; then
    dpkg --add-architecture arm64
  elif [ "$BUILDARCH" = "arm64" ] && [ "$TARGETARCH" = "amd64" ]; then
    dpkg --add-architecture amd64
  fi

  # --- System cleanup & base deps ---
  apt-get update
  apt-get upgrade -y --no-install-recommends

  pkgs=("build-essential")
  pkgs+=("pkg-config")
  pkgs+=("gettext")
  pkgs+=("curl")
  pkgs+=("file")
  pkgs+=("ca-certificates")
  pkgs+=("tar")
  pkgs+=("clang")
  pkgs+=("llvm")
  pkgs+=("lld")

  # Install packages in multiple steps to avoid OOM while dpkg configures packages
  apt-get install -y --no-install-recommends "\${pkgs[@]}"

  pkgs=()
  if [ "$BUILDARCH" = "amd64" ] && [ "$TARGETARCH" = "arm64" ]; then
    pkgs+=("gcc-aarch64-linux-gnu")
    pkgs+=("binutils-aarch64-linux-gnu")
    pkgs+=("libc6-dev-arm64-cross")
    pkgs+=("linux-libc-dev-arm64-cross")
    pkgs+=("libssl-dev:arm64")
    pkgs+=("zlib1g-dev:arm64")
    pkgs+=("libcurl4-openssl-dev:arm64")
    pkgs+=("libexpat1-dev:arm64")
    pkgs+=("libpcre2-dev:arm64")
    pkgs+=("libpcre2-posix3:arm64")
  elif [ "$BUILDARCH" = "arm64" ] && [ "$TARGETARCH" = "amd64" ]; then
    pkgs+=("gcc-x86-64-linux-gnu")
    pkgs+=("binutils-x86-64-linux-gnu")
    pkgs+=("libc6-dev-amd64-cross")
    pkgs+=("linux-libc-dev-amd64-cross")
    pkgs+=("libssl-dev:amd64")
    pkgs+=("zlib1g-dev:amd64")
    pkgs+=("libcurl4-openssl-dev:amd64")
    pkgs+=("libexpat1-dev:amd64")
    pkgs+=("libpcre2-dev:amd64")
    pkgs+=("libpcre2-posix3:amd64")
  else
    pkgs+=("libssl-dev")
    pkgs+=("zlib1g-dev")
    pkgs+=("libcurl4-openssl-dev")
    pkgs+=("libexpat1-dev")
    pkgs+=("libpcre2-dev")
    pkgs+=("libpcre2-posix3")
  fi

  apt-get install -y --no-install-recommends "\${pkgs[@]}"
EOT

WORKDIR /opt/git

ARG GIT_VERSION=2.52.0

RUN <<EOT bash
  set -Eexuo pipefail

  curl -sSfL --tlsv1.3 "https://github.com/git/git/archive/v${GIT_VERSION}.tar.gz" | tar -xzf -

  cd git-${GIT_VERSION}

  make_opts=("prefix=/opt/git")
  make_opts+=("sysconfdir=/etc")
  if [ "$BUILDARCH" = "amd64" ] && [ "$TARGETARCH" = "arm64" ]; then
    make_opts+=("HOST_CPU=x86_64")
    make_opts+=("AR=llvm-ar")
    make_opts+=("RANLIB=llvm-ranlib")
    make_opts+=("STRIP=llvm-strip")
    make_opts+=("CC=clang --target=aarch64-linux-gnu -I/usr/include/aarch64-linux-gnu")
    make_opts+=("CXX=clang++ --target=aarch64-linux-gnu -I/usr/include/aarch64-linux-gnu")
    make_opts+=("CFLAGS=-Os -flto -ffunction-sections -fdata-sections -I/usr/include/aarch64-linux-gnu")
    make_opts+=("CPPFLAGS=-Os -flto -ffunction-sections -fdata-sections -I/usr/include/aarch64-linux-gnu")
    make_opts+=("LDFLAGS=-flto -fuse-ld=lld -Wl,--gc-sections -L/usr/aarch64-linux-gnu")
  elif [ "$BUILDARCH" = "arm64" ] && [ "$TARGETARCH" = "amd64" ]; then
    make_opts+=("HOST_CPU=aarch64")
    make_opts+=("AR=llvm-ar")
    make_opts+=("RANLIB=llvm-ranlib")
    make_opts+=("STRIP=llvm-strip")
    make_opts+=("CC=clang --target=x86_64-linux-gnu -I/usr/include/x86_64-linux-gnu")
    make_opts+=("CXX=clang++ --target=x86_64-linux-gnu -I/usr/include/x86_64-linux-gnu")
    make_opts+=("CFLAGS=-Os -flto -ffunction-sections -fdata-sections -I/usr/include/x86_64-linux-gnu")
    make_opts+=("CPPFLAGS=-Os -flto -ffunction-sections -fdata-sections -I/usr/include/x86_64-linux-gnu")
    make_opts+=("LDFLAGS=-flto -fuse-ld=lld -Wl,--gc-sections -L/usr/x86_64-linux-gnu")
  else
    make_opts+=("AR=llvm-ar")
    make_opts+=("RANLIB=llvm-ranlib")
    make_opts+=("STRIP=llvm-strip")
    make_opts+=("CC=clang")
    make_opts+=("CXX=clang++")
    make_opts+=("CFLAGS=-Os -flto -ffunction-sections -fdata-sections")
    make_opts+=("CPPFLAGS=-Os -flto -ffunction-sections -fdata-sections")
    make_opts+=("LDFLAGS=-flto -fuse-ld=lld -Wl,--gc-sections")
  fi
  make_opts+=("USE_OPENSSL=YesPlease")
  make_opts+=("USE_LIBPCRE=YesPlease")
  make_opts+=("NO_SVN_TESTS=YesPlease")
  make_opts+=("NO_TCLTK=YesPlease")
  make_opts+=("NO_GETTEXT=YesPlease")
  make_opts+=("NO_PERL=YesPlease")
  make_opts+=("NO_PYTHON=YesPlease")
  make_opts+=("NO_GITWEB=YesPlease")

  make -s "\${make_opts[@]}" -j\$(nproc) all
  make -s "\${make_opts[@]}" -j\$(nproc) install

  llvm-strip /opt/git/bin/git
  file /opt/git/bin/git
EOT

FROM docker.io/library/ubuntu:24.04 AS devcontainer

ENV DEBIAN_FRONTEND=noninteractive
ARG USER_UID
ARG USER_GID
ARG TARGETARCH
# https://github.com/bootandy/dust/releases
ARG DUST_VERSION
# https://github.com/sharkdp/bat/releases
ARG BAT_VERSION
# https://github.com/mikefarah/yq/releases
ARG YQ_VERSION
# https://github.com/jqlang/jq/releases
ARG JQ_VERSION
# https://nodejs.org/en/download
ARG NODE_VERSION

WORKDIR /root

RUN <<EOT bash
  set -Eexuo pipefail

  # --- System cleanup & base deps ---
  apt-get update
  apt-get upgrade -y --no-install-recommends
  apt-get install -y --no-install-recommends \
    apt-transport-https \
    apt-utils \
    bash-completion \
    btop \
    ca-certificates \
    coreutils \
    curl \
    direnv \
    dnsutils \
    fd-find \
    file \
    fzf \
    gnupg2 \
    gosu \
    httpie \
    iputils-ping \
    less \
    libicu74 \
    locales \
    net-tools \
    netcat-openbsd \
    nload \
    openssh-client \
    procps \
    ripgrep \
    sed \
    software-properties-common \
    tree \
    unzip \
    util-linux \
    wget \
    xz-utils \
    zip

  # --- Locale ---
  localedef -i en_US -c -f UTF-8 en_US.UTF-8

  update-ca-certificates

  # --- User ---
  if [ '1000' = "${USER_UID}" ]; then
    usermod -l vscode ubuntu
    groupmod -n vscode ubuntu
    usermod -d /home/vscode -m vscode
    usermod -G vscode vscode
  else
    groupadd --gid "${USER_GID}" vscode
    useradd -m -u "${USER_UID}" -g "${USER_GID}" -s /bin/bash vscode
  fi

  # --- Bash completion dirs ---
  gosu vscode bash -c 'mkdir -p /home/vscode/.local/share/bash-completion/completions'

  # --- Install dust ---
  case "${TARGETARCH}" in
    amd64) DUST_ARCH=x86_64 ;;
    arm64) DUST_ARCH=aarch64 ;;
    *) echo "Unsupported target platform architecture: ${TARGETARCH}" >&2; exit 16 ;;
  esac;
  curl -fsSL --tlsv1.3 "https://github.com/bootandy/dust/releases/download/${DUST_VERSION}/dust-${DUST_VERSION}-\${DUST_ARCH}-unknown-linux-gnu.tar.gz" | tar -xzf - --strip-components=1 -C /usr/local/bin/ "dust-${DUST_VERSION}-\${DUST_ARCH}-unknown-linux-gnu/dust"
  gosu vscode dust --version

  # --- Install bat ---
  case "${TARGETARCH}" in
    amd64) BAT_ARCH=x86_64 ;;
    arm64) BAT_ARCH=aarch64 ;;
    *) echo "Unsupported target platform architecture: ${TARGETARCH}" >&2; exit 16 ;;
  esac;
  mkdir /tmp/bat
  curl -fsSL --tlsv1.3 "https://github.com/sharkdp/bat/releases/download/${BAT_VERSION}/bat-${BAT_VERSION}-\${BAT_ARCH}-unknown-linux-gnu.tar.gz" | tar -xzf - --strip-components=1 -C /tmp/bat "bat-${BAT_VERSION}-\${BAT_ARCH}-unknown-linux-gnu"
  mv /tmp/bat/bat /usr/local/bin/bat
  mv /tmp/bat/autocomplete/bat.bash /home/vscode/.local/share/bash-completion/completions/bat.bash
  rm -rf /tmp/bat
  chown vscode:vscode /home/vscode/.local/share/bash-completion/completions/bat.bash
  gosu vscode bat --version

  # --- Install jq ---
  curl -fsSL --tlsv1.3 "https://github.com/jqlang/jq/releases/download/jq-${JQ_VERSION}/jq-linux-${TARGETARCH}" -o /usr/local/bin/jq
  chmod +x /usr/local/bin/jq
  gosu vscode jq --version

  # --- Install yq ---
  curl -fsSL --tlsv1.3 "https://github.com/mikefarah/yq/releases/download/${YQ_VERSION}/yq_linux_${TARGETARCH}" -o /usr/local/bin/yq
  chmod +x /usr/local/bin/yq
  gosu vscode yq --version

  # --- Install Node ---
  if [ "$TARGETARCH" = "amd64" ]; then
    NODE_ARCH="x64"
  elif [ "$TARGETARCH" = "arm64" ]; then
    NODE_ARCH="arm64"
  else
    echo "Unsupported target platform architecture: ${TARGETARCH}"
    exit 16
  fi
  mkdir /opt/node
  curl -fsSLO --tlsv1.3 "https://nodejs.org/dist/${NODE_VERSION}/node-${NODE_VERSION}-linux-\$NODE_ARCH.tar.xz"
  curl -fsSL --tlsv1.3 -o /tmp/nodejs-keyring.kbx https://github.com/nodejs/release-keys/raw/HEAD/gpg/pubring.kbx
  curl -fsSLO --tlsv1.3 "https://nodejs.org/dist/${NODE_VERSION}/SHASUMS256.txt.asc"
  gpgv --keyring=/tmp/nodejs-keyring.kbx --output SHASUMS256.txt < SHASUMS256.txt.asc
  sha256sum --check SHASUMS256.txt --ignore-missing
  tar -xJf "node-${NODE_VERSION}-linux-\$NODE_ARCH.tar.xz" -C /opt/node --strip-components=1
  rm /tmp/nodejs-keyring.kbx SHASUMS256.txt.asc SHASUMS256.txt "node-${NODE_VERSION}-linux-\$NODE_ARCH.tar.xz"
  gosu vscode /opt/node/bin/node --version
  gosu vscode bash -c '/opt/node/bin/node --completion-bash > /home/vscode/.local/share/bash-completion/completions/node.bash'
  gosu vscode /opt/node/bin/npm --version
  gosu vscode bash -c '/opt/node/bin/npm completion bash > /home/vscode/.local/share/bash-completion/completions/npm.bash'

  # --- Cleanup ---
  apt-get purge -y gosu
  apt-get autoremove -y
  apt-get clean
  rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
EOT

ENV LANG=en_US.utf8 LC_ALL=en_US.utf8 PATH="$PATH:/opt/node/bin:/opt/git/bin"

COPY --chown=vscode:vscode --from=git-build /opt/git/share/bash-completion/completions/git /home/vscode/.local/share/bash-completion/completions/git.bash
COPY --chown=root:root --from=git-build /opt/git /opt/git
RUN git --version

USER vscode
WORKDIR /workspaces/taiga-front-next
