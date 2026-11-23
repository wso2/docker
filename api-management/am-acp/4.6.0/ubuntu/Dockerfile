# API Control Plane 4.6.0 image, on Ubuntu. See README.md in the product directory
# for how to build it.

# The product version to build is passed with
# --build-arg PRODUCT_VERSION=4.6.0.<update level>.
ARG BINARY_REPOSITORY=registry.wso2.com/wso2-apim/binaries/am-acp
ARG PRODUCT_VERSION

# Downloads the product binary. Your registry credentials are used only here, and
# never reach the image.
FROM --platform=$BUILDPLATFORM ghcr.io/oras-project/oras:v1.3.6@sha256:9cfb578be975b762fd898b9d004243d8003aeb807a3e15b2ed212cdd1326c9eb AS fetch
ARG BINARY_REPOSITORY
ARG PRODUCT_VERSION
RUN --mount=type=secret,id=REGISTRY_USERNAME \
    --mount=type=secret,id=REGISTRY_PASSWORD \
    set -eu; \
    if [ -z "${PRODUCT_VERSION:-}" ]; then \
        echo "Error: pass the product version, e.g. --build-arg PRODUCT_VERSION=4.6.0.26" >&2; \
        exit 1; \
    fi; \
    if [ ! -s /run/secrets/REGISTRY_USERNAME ] || [ ! -s /run/secrets/REGISTRY_PASSWORD ]; then \
        echo "Error: pass your registry credentials, e.g. --secret id=REGISTRY_USERNAME --secret id=REGISTRY_PASSWORD" >&2; \
        exit 1; \
    fi; \
    oras login "${BINARY_REPOSITORY%%/*}" -u "$(cat /run/secrets/REGISTRY_USERNAME)" \
        --password-stdin < /run/secrets/REGISTRY_PASSWORD; \
    oras pull "${BINARY_REPOSITORY}:${PRODUCT_VERSION}" -o /tmp/pull; \
    set -- /tmp/pull/*.zip; \
    if [ "$#" -ne 1 ] || [ ! -f "$1" ]; then \
        echo "Error: expected one zip in ${BINARY_REPOSITORY}:${PRODUCT_VERSION}" >&2; \
        exit 1; \
    fi; \
    mkdir /binary; \
    unzip -q "$1" -d /binary; \
    rm -rf /tmp/pull "${HOME}/.docker"

FROM scratch AS binary
COPY --from=fetch /binary/ /

# set base Docker image to Ubuntu
FROM ubuntu:24.04

ENV LANG='en_US.UTF-8' LANGUAGE='en_US:en' LC_ALL='en_US.UTF-8'

# install dependencies
RUN apt-get update \
    && DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends tzdata curl ca-certificates fontconfig locales libxml2-utils netcat-traditional \
    && echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen \
    && locale-gen en_US.UTF-8 \
    && rm -rf /var/lib/apt/lists/*

ENV JAVA_VERSION=jdk-21.0.13+9
ENV JAVA_HOME=/opt/java/openjdk \
    PATH="/opt/java/openjdk/bin:$PATH"

# install Temurin OpenJDK 21
RUN set -eux; \
    ARCH="$(dpkg --print-architecture)"; \
    case "${ARCH}" in \
       amd64) \
         ESUM='d7f5d5cf865a2902b70ff549b97586a7d3f234e016f041ab447ea740c7d66d1c'; \
         BINARY_URL='https://github.com/adoptium/temurin21-binaries/releases/download/jdk-21.0.13%2B9/OpenJDK21U-jdk_x64_linux_hotspot_21.0.13_9.tar.gz'; \
         ;; \
       arm64) \
         ESUM='e3522e707fc7dbe2a72c30746b1e7068301a0213ecd213d4491f35a14eb55578'; \
         BINARY_URL='https://github.com/adoptium/temurin21-binaries/releases/download/jdk-21.0.13%2B9/OpenJDK21U-jdk_aarch64_linux_hotspot_21.0.13_9.tar.gz'; \
         ;; \
       ppc64el) \
         ESUM='cfaedcc659213833ead9c2a50987d8dcbd0e7a2d82217abb43acb64c0dd3d8cc'; \
         BINARY_URL='https://github.com/adoptium/temurin21-binaries/releases/download/jdk-21.0.13%2B9/OpenJDK21U-jdk_ppc64le_linux_hotspot_21.0.13_9.tar.gz'; \
         ;; \
       s390x) \
         ESUM='597271ef81f7e4d0e2b86aa6e4d484af194253bdf238908a8394b78970dff713'; \
         BINARY_URL='https://github.com/adoptium/temurin21-binaries/releases/download/jdk-21.0.13%2B9/OpenJDK21U-jdk_s390x_linux_hotspot_21.0.13_9.tar.gz'; \
         ;; \
       *) \
         echo "Unsupported arch: ${ARCH}"; \
         exit 1; \
         ;; \
    esac; \
    curl -LfsSo /tmp/openjdk.tar.gz ${BINARY_URL}; \
    echo "${ESUM} */tmp/openjdk.tar.gz" | sha256sum -c -; \
    mkdir -p /opt/java/openjdk; \
    cd /opt/java/openjdk; \
    tar -xf /tmp/openjdk.tar.gz --strip-components=1; \
    rm -rf /tmp/openjdk.tar.gz; \
    java -Xshare:dump;

LABEL maintainer="WSO2 Docker Maintainers <dev@wso2.org>" \
      com.wso2.docker.source="https://github.com/wso2/docker-apim/releases/tag/v4.6.0.1"

# set Docker image build arguments
# build arguments for user/group configurations
ARG USER=wso2carbon
ARG USER_ID=10001
ARG USER_GROUP=wso2
ARG USER_GROUP_ID=10001
ARG USER_HOME=/home/${USER}
# build arguments for WSO2 product installation
ARG WSO2_SERVER_NAME=wso2am-acp
ARG WSO2_SERVER_VERSION=4.6.0
ARG WSO2_SERVER=${WSO2_SERVER_NAME}-${WSO2_SERVER_VERSION}
ARG WSO2_SERVER_HOME=${USER_HOME}/${WSO2_SERVER}
# build argument for MOTD
ARG MOTD="\n\
Welcome to WSO2 Docker resources.\n\
------------------------------------ \n\
This Docker container comprises of a WSO2 product, running with its latest GA release \n\
which is under the Apache License, Version 2.0. \n\
Read more about Apache License, Version 2.0 here @ http://www.apache.org/licenses/LICENSE-2.0.\n"

# create the non-root user and group and set MOTD login message
RUN \
    groupadd --system -g ${USER_GROUP_ID} ${USER_GROUP} \
    && useradd --system --create-home --home-dir ${USER_HOME} --no-log-init -g ${USER_GROUP_ID} -u ${USER_ID} ${USER} \
    && echo '[ ! -z "${TERM}" -a -r /etc/motd ] && cat /etc/motd' >> /etc/bash.bashrc; echo "${MOTD}" > /etc/motd

# copy init script to user home
COPY --chown=wso2carbon:wso2 docker-entrypoint.sh ${USER_HOME}/
# install required packages

# add the WSO2 product distribution to user's home directory
COPY --from=binary --chown=wso2carbon:wso2 /${WSO2_SERVER}/ ${WSO2_SERVER_HOME}/
RUN \
    mkdir ${USER_HOME}/wso2-tmp \
    && bash -c 'mkdir -p ${USER_HOME}/solr/{indexed-data,database}' \
    && chown wso2carbon:wso2 -R ${USER_HOME}/solr \
    && cp -r ${WSO2_SERVER_HOME}/repository/deployment/server/executionplans ${USER_HOME}/wso2-tmp

# remove unnecesary packages
RUN apt-get purge -y netcat-traditional

# set the user and work directory
USER ${USER_ID}
WORKDIR ${USER_HOME}

# set environment variables
ENV WORKING_DIRECTORY=${USER_HOME} \
    WSO2_SERVER_HOME=${WSO2_SERVER_HOME}

# expose ports
EXPOSE 9763 9443 9999 11111 8280 8243 5672 9711 9611 9099

# initiate container and start WSO2 Carbon server
ENTRYPOINT ["/home/wso2carbon/docker-entrypoint.sh"]
