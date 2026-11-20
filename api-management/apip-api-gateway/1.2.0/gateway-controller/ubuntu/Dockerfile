# syntax=docker/dockerfile:1
# Build from the registry product ZIP; see ../../../README.md for local builds.
ARG BASE_IMAGE=ubuntu:24.04
ARG BINARY_REPOSITORY=registry.wso2.com/wso2-api-platform/binaries/api-gateway
ARG PRODUCT_VERSION

# Registry credentials are available only to this build step.
FROM --platform=$BUILDPLATFORM ghcr.io/oras-project/oras:v1.3.6@sha256:9cfb578be975b762fd898b9d004243d8003aeb807a3e15b2ed212cdd1326c9eb AS fetch
ARG BINARY_REPOSITORY
ARG PRODUCT_VERSION
RUN --mount=type=secret,id=REGISTRY_USERNAME \
    --mount=type=secret,id=REGISTRY_PASSWORD \
    set -eu; \
    if [ -z "${PRODUCT_VERSION:-}" ]; then \
        echo "Error: pass --build-arg PRODUCT_VERSION=1.2.0.<update level>" >&2; \
        exit 1; \
    fi; \
    if [ ! -s /run/secrets/REGISTRY_USERNAME ] || [ ! -s /run/secrets/REGISTRY_PASSWORD ]; then \
        echo "Error: pass --secret id=REGISTRY_USERNAME --secret id=REGISTRY_PASSWORD" >&2; \
        exit 1; \
    fi; \
    oras login "${BINARY_REPOSITORY%%/*}" -u "$(cat /run/secrets/REGISTRY_USERNAME)" \
        --password-stdin < /run/secrets/REGISTRY_PASSWORD; \
    oras pull "${BINARY_REPOSITORY}:${PRODUCT_VERSION}" -o /tmp/pull; \
    set -- /tmp/pull/*.zip; \
    if [ "$#" -ne 1 ] || [ ! -f "$1" ]; then \
        echo "Error: expected one ZIP in ${BINARY_REPOSITORY}:${PRODUCT_VERSION}" >&2; \
        exit 1; \
    fi; \
    mkdir /binary; \
    unzip -q "$1" -d /binary; \
    test -d "/binary/api-gateway.${PRODUCT_VERSION}/gateway-controller" && \
    test -d "/binary/api-gateway.${PRODUCT_VERSION}/gateway-runtime" || { \
        echo "Error: expected api-gateway.${PRODUCT_VERSION}/{gateway-controller,gateway-runtime} in ZIP" >&2; \
        exit 1; \
    }; \
    rm -rf /tmp/pull "${HOME}/.docker"

# Override with --build-context binary=<extracted product directory> for local builds.
FROM scratch AS binary
ARG PRODUCT_VERSION
COPY --from=fetch /binary/api-gateway.${PRODUCT_VERSION}/ /

FROM ${BASE_IMAGE} AS production
ARG TARGETARCH
ARG PRODUCT_VERSION
ARG VERSION=${PRODUCT_VERSION:-unknown}
ARG REVISION=unknown

RUN case "$TARGETARCH" in amd64|arm64) ;; *) echo "Unsupported architecture: $TARGETARCH" >&2; exit 1 ;; esac && \
    apt-get update && \
    apt-get upgrade -y && \
    apt-get install -y --no-install-recommends bash coreutils ca-certificates libsqlite3-0 && \
    rm -rf /var/lib/apt/lists/* && \
    groupadd --system --gid 10001 wso2 && \
    useradd --system --uid 10001 --gid wso2 --home-dir /home/wso2 --shell /usr/sbin/nologin wso2 && \
    mkdir -p /app/data /coverage /home/wso2 && \
    chown -R 10001:0 /app/data /coverage /home/wso2 && \
    chmod -R g=u /app/data /coverage /home/wso2

WORKDIR /app
COPY --from=binary --chmod=0755 /gateway-controller/controller-${TARGETARCH} /app/controller
COPY --from=binary /gateway-controller/default-policies/ /app/default-policies/
COPY --from=binary /gateway-controller/default-llm-provider-templates/ /app/default-llm-provider-templates/
COPY --from=binary /gateway-controller/lua/ /app/lua/
COPY --from=binary /gateway-controller/build-manifest.yaml /gateway-controller/LICENSE /app/
COPY --from=binary --chmod=0755 /gateway-controller/health-check.sh /usr/local/bin/health-check.sh

ENV HOME=/home/wso2
USER 10001:0
EXPOSE 9090 9092 18000 18001
LABEL org.opencontainers.image.title="WSO2 API Platform Gateway Controller" \
      org.opencontainers.image.vendor="WSO2" \
      org.opencontainers.image.version="${VERSION}" \
      org.opencontainers.image.revision="${REVISION}" \
      org.opencontainers.image.licenses="Apache-2.0"
HEALTHCHECK --interval=30s --timeout=5s --start-period=30s --retries=3 \
    CMD ["/usr/local/bin/health-check.sh"]
ENTRYPOINT ["/app/controller"]
