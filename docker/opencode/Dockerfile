FROM debian:bookworm-slim
# Install dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl ca-certificates bash passwd \
    && rm -rf /var/lib/apt/lists/*

# Corporate TLS-intercepting proxies re-sign HTTPS with a root CA the host
# trusts but this build does not, breaking the installer download below. Trust
# any PEM dropped into certs/ (the Windows launcher mirrors docker/certs/ here;
# see docker/certs/README.md). No-op when the directory is empty.
COPY certs/ /tmp/om-certs/
RUN set -eu; \
    for cert in /tmp/om-certs/*.crt /tmp/om-certs/*.pem; do \
        [ -f "$cert" ] || continue; \
        name="$(basename "$cert")"; \
        cp "$cert" "/usr/local/share/ca-certificates/om-extra-${name%.*}.crt"; \
    done; \
    update-ca-certificates; \
    rm -rf /tmp/om-certs

# Install OpenCode and move to system path. The installer resolves "latest"
# through the unauthenticated GitHub API, which rate-limits shared/corporate
# IPs and fails transiently ("Failed to fetch version information") - so retry
# the install, and let OPENCODE_VERSION pin an exact version, which skips the
# API lookup entirely: docker compose build --build-arg OPENCODE_VERSION=x.y.z
ARG OPENCODE_VERSION=""
RUN set -eu; \
    curl -fsSL --retry 3 --retry-delay 5 https://opencode.ai/install -o /tmp/opencode-install.sh; \
    install_args=""; \
    if [ -n "$OPENCODE_VERSION" ]; then install_args="--version $OPENCODE_VERSION"; fi; \
    attempt=1; \
    until bash /tmp/opencode-install.sh $install_args; do \
        if [ "$attempt" -ge 3 ]; then echo "OpenCode install failed after $attempt attempts" >&2; exit 1; fi; \
        attempt=$((attempt + 1)); \
        echo "OpenCode install failed - retrying (attempt $attempt of 3)..." >&2; \
        sleep 10; \
    done; \
    rm -f /tmp/opencode-install.sh; \
    cp /root/.opencode/bin/opencode /usr/local/bin/opencode; \
    chmod +x /usr/local/bin/opencode

# Create non-root user
RUN useradd -m -s /bin/bash opencode

# Switch to non-root user
USER opencode
WORKDIR /home/opencode

# Create config directory
RUN mkdir -p /home/opencode/.config/opencode

# Copy instructions to working directory (OpenCode resolves relative to $CWD)
COPY --chown=opencode:opencode AGENTS.md /home/opencode/AGENTS.md
COPY --chown=opencode:opencode entrypoint.sh /home/opencode/entrypoint.sh

USER root
RUN chmod +x /home/opencode/entrypoint.sh
USER opencode

EXPOSE 4096

# Use entrypoint to generate config dynamically
ENTRYPOINT ["/home/opencode/entrypoint.sh"]
