# Universal Gateway 4.7.0 image, on Alpine. See README.md in the product directory
# for how to build it.

# The product version to build is passed with
# --build-arg PRODUCT_VERSION=4.7.0.<update level>.
ARG BINARY_REPOSITORY=registry.wso2.com/wso2-apim/binaries/am-universal-gw
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
        echo "Error: pass the product version, e.g. --build-arg PRODUCT_VERSION=4.7.0.26" >&2; \
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

# set base Docker image to Alpine
FROM alpine:3.23

ENV LANG='en_US.UTF-8' LANGUAGE='en_US:en' LC_ALL='en_US.UTF-8'

# install dependencies
RUN apk add --no-cache tzdata musl-locales musl-locales-lang bash libxml2-utils netcat-openbsd \
    && rm -rf /var/cache/apk/*

ENV JAVA_VERSION=jdk-25.0.5+8
ENV JAVA_HOME=/opt/java/openjdk \
    PATH="/opt/java/openjdk/bin:$PATH"

# install Temurin OpenJDK 25
RUN set -eux; \
    ARCH="$(apk --print-arch)"; \
    case "${ARCH}" in \
       aarch64) \
         ESUM='d02eadc0a0f93faba4be38c8657eb8eef842103a4c7c2400f5827dbd90fc21c9'; \
         BINARY_URL='https://github.com/adoptium/temurin25-binaries/releases/download/jdk-25.0.5%2B8/OpenJDK25U-jdk_aarch64_alpine-linux_hotspot_25.0.5_8.tar.gz'; \
         ;; \
       x86_64) \
         ESUM='efc86581260b73c340c313e7577c4c85d6a8a899edc3748d78bee5238fe44a4e'; \
         BINARY_URL='https://github.com/adoptium/temurin25-binaries/releases/download/jdk-25.0.5%2B8/OpenJDK25U-jdk_x64_alpine-linux_hotspot_25.0.5_8.tar.gz'; \
         ;; \
       *) \
         echo "Unsupported arch: ${ARCH}"; \
         exit 1; \
         ;; \
    esac; \
    wget -O /tmp/openjdk.tar.gz ${BINARY_URL}; \
    echo "${ESUM} */tmp/openjdk.tar.gz" | sha256sum -c -; \
    mkdir -p /opt/java/openjdk; \
    tar --extract \
        --file /tmp/openjdk.tar.gz \
        --directory /opt/java/openjdk \
        --strip-components 1 \
        --no-same-owner \
    ; \
    rm -rf /tmp/openjdk.tar.gz;


LABEL maintainer="WSO2 Docker Maintainers <dev@wso2.org>"  \
      com.wso2.docker.source="https://github.com/wso2/docker-apim/releases/tag/v4.7.0.1"

# set Docker image build arguments
# build arguments for user/group configurations
ARG USER=wso2carbon
ARG USER_ID=10001
ARG USER_GROUP=wso2
ARG USER_GROUP_ID=10001
ARG USER_HOME=/home/${USER}
# build arguments for WSO2 product installation
ARG WSO2_SERVER_NAME=wso2am-universal-gw
ARG WSO2_SERVER_VERSION=4.7.0
ARG WSO2_SERVER=${WSO2_SERVER_NAME}-${WSO2_SERVER_VERSION}
ARG WSO2_SERVER_HOME=${USER_HOME}/${WSO2_SERVER}
# build argument for MOTD
ARG MOTD='printf "\n\
 Welcome to WSO2 Docker Resources \n\
 --------------------------------- \n\
 This Docker container comprises of a WSO2 product, running with its latest GA release \n\
 which is under the Apache License, Version 2.0. \n\
 Read more about Apache License, Version 2.0 here @ http://www.apache.org/licenses/LICENSE-2.0.\n"'
ENV ENV=${USER_HOME}"/.ashrc"

# create the non-root user and group and set MOTD login message
RUN \
    addgroup -S -g ${USER_GROUP_ID} ${USER_GROUP} \
    && adduser -S -u ${USER_ID} -h ${USER_HOME} -G ${USER_GROUP} ${USER} \
    && echo ${MOTD} > "${ENV}"

# copy init script to user home
COPY --chown=wso2carbon:wso2 docker-entrypoint.sh ${USER_HOME}/
    
# add the WSO2 product distribution to user's home directory
COPY --from=binary --chown=wso2carbon:wso2 /${WSO2_SERVER}/ ${WSO2_SERVER_HOME}/
RUN \
    mkdir ${USER_HOME}/wso2-tmp \
    && bash -c 'mkdir -p ${USER_HOME}/solr/{indexed-data,database}' \
    && chown wso2carbon:wso2 -R ${USER_HOME}/solr \
    && cp -r ${WSO2_SERVER_HOME}/repository/deployment/server/synapse-configs ${USER_HOME}/wso2-tmp

# remove unnecesary packages
RUN apk del netcat-openbsd

# set the user and work directory
USER ${USER_ID}
WORKDIR ${USER_HOME}

# set environment variables
ENV WORKING_DIRECTORY=${USER_HOME} \
    WSO2_SERVER_HOME=${WSO2_SERVER_HOME}

# expose ports
EXPOSE 8280 8243 9763 9443 9099 8099 9021 8021

# initiate container and start WSO2 Carbon server
ENTRYPOINT ["/home/wso2carbon/docker-entrypoint.sh"]
