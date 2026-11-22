# Universal Gateway

Dockerfiles for the WSO2 API Manager Universal Gateway images, in one directory per product
version and base OS:

| Version | Alpine | Ubuntu | Rocky Linux |
|---|---|---|---|
| 4.7.0 | `4.7.0/alpine` | `4.7.0/ubuntu` | `4.7.0/rocky` |
| 4.6.0 | `4.6.0/alpine` | `4.6.0/ubuntu` | `4.6.0/rocky` |

Each build downloads the updated product pack of the product version you pass,
`<version>.<update level>` (for example `4.7.0.26`), from
`registry.wso2.com/wso2-apim/binaries/am-universal-gw` with ORAS. Apart from that download,
each Dockerfile matches the one the released images of its version are built
from.

## Building an image

### Step 1: Retrieve a registry token

Generate a registry token as described in
[Registry Token Management](https://updates.docs.choreolabs.com/en/latest/updates/registry-token-management-customer/).

### Step 2: Set your credentials

```bash
export REGISTRY_USERNAME='<token-name>'
export REGISTRY_PASSWORD='<token-secret>'
```

### Step 3: Build the image

Pass the product version to build, and the directory that holds the Dockerfile
for that version (for example `4.7.0/ubuntu` for `4.7.0.26`, or `4.6.0/ubuntu`
for a 4.6.0 update level). Run the command from this directory:

```bash
docker build \
    --build-arg PRODUCT_VERSION=<product-version> \
    --secret id=REGISTRY_USERNAME \
    --secret id=REGISTRY_PASSWORD \
    -t <user-registry>/<user-image>:<user-image-version> <dockerfile-dir>
```

For example, to build the Ubuntu image of product version 4.7.0.26:

```bash
docker build \
    --build-arg PRODUCT_VERSION=4.7.0.26 \
    --secret id=REGISTRY_USERNAME \
    --secret id=REGISTRY_PASSWORD \
    -t registry.example.com/am-universal-gw:4.7.0.26 4.7.0/ubuntu
```

### Both architectures

To build for `linux/amd64` and `linux/arm64` together, use `docker buildx build`
with the same arguments. Docker's default image store keeps one architecture per
image, so push the result straight to your registry:

```bash
docker buildx build --platform linux/amd64,linux/arm64 \
    --build-arg PRODUCT_VERSION=<product-version> \
    --secret id=REGISTRY_USERNAME \
    --secret id=REGISTRY_PASSWORD \
    -t <user-registry>/<user-image>:<user-image-version> --push <dockerfile-dir>
```

## Changing the base image

Edit the `FROM` line after the `binary` stage, and keep the image's runtime
contract (user, ports, paths, and entrypoint) unchanged. The base must be able to
run the Temurin JDK the Dockerfile installs (a musl build on Alpine, a glibc
build on the others), and provide `bash`, which the entrypoint uses. Replace the
package manager, `groupadd` and `useradd` commands with your base's equivalents,
and keep UID 10001.

## Requirements

Docker with BuildKit, which is the default since Docker 23.0.
