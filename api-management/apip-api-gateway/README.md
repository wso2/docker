# API Gateway

Dockerfiles for the WSO2 API Platform API Gateway 1.2.0 images:

| Component | Dockerfile directory | Default base image | Exposed ports |
| --- | --- | --- | --- |
| Gateway Controller | `1.2.0/gateway-controller/ubuntu` | `ubuntu:24.04` | `9090`, `9092`, `18000`, `18001` |
| Gateway Runtime | `1.2.0/gateway-runtime/ubuntu` | `ubuntu:22.04` | `8080`, `8443`, `9901`, `9002`, `9003` |

Run the commands below from this directory. Both images support `linux/amd64`
and `linux/arm64` and run as UID 10001 with GID 0.

## Requirements

- Docker with BuildKit and Buildx support.
- For registry builds, a registry token with access to the API Gateway binary
  repository.
- The full product version to build, in the form `1.2.0.<update level>`.

## Building from the registry

### Step 1: Retrieve a registry token

Generate a registry token as described in
[Registry Token Management](https://updates.docs.choreolabs.com/en/latest/updates/registry-token-management-customer/).
Use the token name as `REGISTRY_USERNAME` and the token secret as
`REGISTRY_PASSWORD`.

### Step 2: Set your credentials

```bash
export REGISTRY_USERNAME='<token-name>'
export REGISTRY_PASSWORD='<token-secret>'
```

### Step 3: Build the images

Set the product version, replacing the placeholder with an available registry
tag, then build both images:

```bash
export PRODUCT_VERSION='1.2.0.<update level>'

docker build \
    --build-arg PRODUCT_VERSION="$PRODUCT_VERSION" \
    --secret id=REGISTRY_USERNAME \
    --secret id=REGISTRY_PASSWORD \
    -t "api-gateway-controller:$PRODUCT_VERSION" \
    1.2.0/gateway-controller/ubuntu

docker build \
    --build-arg PRODUCT_VERSION="$PRODUCT_VERSION" \
    --secret id=REGISTRY_USERNAME \
    --secret id=REGISTRY_PASSWORD \
    -t "api-gateway-runtime:$PRODUCT_VERSION" \
    1.2.0/gateway-runtime/ubuntu
```

Each build pulls the product ZIP from
`registry.wso2.com/wso2-api-platform/binaries/api-gateway:$PRODUCT_VERSION`.
The artifact must contain exactly one ZIP, with both component directories under
`api-gateway.$PRODUCT_VERSION/`. Credentials are passed as BuildKit secrets to the
fetch step.

## Building from local binaries

Extract the product ZIP and supply its product directory as the named `binary`
build context. It must contain `gateway-controller/` and `gateway-runtime/`,
including the binaries for the target architecture and their supporting files.
This overrides the binary stage and skips the registry fetch; registry
credentials are not required. Package installation still requires network access.

```bash
export PRODUCT_VERSION='1.2.0.<update level>'
export PRODUCT_DIR="/absolute/path/to/api-gateway.$PRODUCT_VERSION"

docker buildx build --load \
    --build-context "binary=$PRODUCT_DIR" \
    --build-arg PRODUCT_VERSION="$PRODUCT_VERSION" \
    -t "api-gateway-controller:$PRODUCT_VERSION" \
    1.2.0/gateway-controller/ubuntu

docker buildx build --load \
    --build-context "binary=$PRODUCT_DIR" \
    --build-arg PRODUCT_VERSION="$PRODUCT_VERSION" \
    -t "api-gateway-runtime:$PRODUCT_VERSION" \
    1.2.0/gateway-runtime/ubuntu
```

## Building both architectures

Use a Buildx builder configured for both target platforms. Repeat for each
component, using its Dockerfile directory and image name:

```bash
docker buildx build --platform linux/amd64,linux/arm64 \
    --build-arg PRODUCT_VERSION="$PRODUCT_VERSION" \
    --secret id=REGISTRY_USERNAME \
    --secret id=REGISTRY_PASSWORD \
    -t "registry.example.com/api-gateway-controller:$PRODUCT_VERSION" \
    --push 1.2.0/gateway-controller/ubuntu
```

Replace `registry.example.com` with your registry and log in before pushing.
For local binary builds, replace the two secret options with
`--build-context "binary=$PRODUCT_DIR"`.

## Build arguments

| Argument | Purpose |
| --- | --- |
| `PRODUCT_VERSION` | Product ZIP tag and extracted directory version; required for registry builds. Also supplies the default image version label. |
| `BINARY_REPOSITORY` | Override the binary repository, without a tag. |
| `BASE_IMAGE` | Override the component's Ubuntu base image. |
| `VERSION` | Override the OCI image version label. |
| `REVISION` | Set the OCI image revision label; defaults to `unknown`. |

## Changing the base image

Pass `--build-arg BASE_IMAGE=<image>:<tag>`. The Dockerfiles use `apt-get`,
`groupadd`, and `useradd`, so other distributions require Dockerfile changes.
Preserve UID 10001, writable paths, ports, health checks, and entrypoints.

- **Gateway Controller:** requires a glibc-based image and `libsqlite3-0`.
- **Gateway Runtime:** requires compatibility with the supplied Envoy binary and
  Python dependencies. The same base is used for the Python dependency builder
  and production image. The build checks that Envoy runs and Python can import
  `grpc` and `google.protobuf`.

## Runtime configuration

The controller starts `/app/controller`. The runtime starts its supplied
`docker-entrypoint.sh` through Tini and defaults to
`GATEWAY_CONTROLLER_HOST=gateway-controller` and `LOG_LEVEL=info`. Provide a
network-resolvable controller hostname, or override `GATEWAY_CONTROLLER_HOST`.

These Dockerfiles build the images; they do not supply a complete gateway
deployment configuration. Use the configuration and deployment instructions
distributed with your product version. Exposed ports are image metadata and
must be published explicitly when host access is needed. Any mounted writable
directories must allow access by UID 10001 or GID 0.
