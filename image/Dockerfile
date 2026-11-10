# omp agent box: omp + tmux (+ optional loopback sshd) for a long-lived pod.
FROM debian:trixie-slim

ARG OMP_VERSION=18.18.6
ARG AGENT_UID=1000

LABEL org.opencontainers.image.source=https://github.com/M0Rf30/omp-kube \
      org.opencontainers.image.description="oh-my-pi coding agent box for Kubernetes" \
      org.opencontainers.image.licenses=MIT

ENV LANG=C.UTF-8 \
    LC_ALL=C.UTF-8 \
    TERM=xterm-256color

RUN apt-get update \
 && apt-get install -y --no-install-recommends \
      bash ca-certificates curl git jq less openssh-client openssh-server \
      procps python3 ripgrep tini tmux unzip xz-utils \
 && rm -rf /var/lib/apt/lists/* \
 && rm -f /etc/ssh/ssh_host_* \
 && install -d -m 0755 /run/sshd

# Prebuilt binary, verified against the build service's sha256 by the installer.
RUN curl -fsSL https://omp.sh/install \
      | PI_INSTALL_DIR=/usr/local/bin sh -s -- --binary --ref "v${OMP_VERSION}" \
 && rm -rf /root/.omp

RUN useradd --uid "${AGENT_UID}" --user-group --create-home --home-dir /home/agent --shell /bin/bash agent

COPY --chmod=0755 omp-pod /usr/local/bin/omp-pod

USER ${AGENT_UID}
WORKDIR /home/agent
ENTRYPOINT ["/usr/bin/tini", "--", "/usr/local/bin/omp-pod"]
