# FROM openjdk:11-jre-slim

ARG base_version

FROM containers.instana.io/instana/product/jdk11:$base_version

ARG commit_id
ARG vendor_version
ARG image_version
ARG bcfips_version
ARG TARGETARCH

LABEL com.instana.commit.id=$commit_id
LABEL com.instana.vendor.version=$vendor_version
LABEL com.instana.image.version=$image_version
LABEL com.instana.group=databases
LABEL com.instana.name=zookeeper

ENV PATH="$PATH:/opt/instana/zookeeper/bin"
ENV ZOO_FIPS_DIR="/etc/instana/zookeeper"

ENV ZOO_CONF_DIR=/conf \
    ZOO_DATA_DIR=/data \
    ZOO_DATA_LOG_DIR=/datalog \
    ZOO_LOG_DIR=/logs \
    ZOO_TICK_TIME=2000 \
    ZOO_INIT_LIMIT=10 \
    ZOO_SYNC_LIMIT=5 \
    ZOO_AUTOPURGE_PURGEINTERVAL=1 \
    ZOO_AUTOPURGE_SNAPRETAINCOUNT=20 \
    ZOO_MAX_CLIENT_CNXNS=60 \
    ZOO_STANDALONE_ENABLED=true \
    ZOO_ADMINSERVER_ENABLED=false \
    ZOO_CLIENT_PORT=2181

COPY --link health_zookeeper_$TARGETARCH /health
COPY --link assets/ "$ZOO_FIPS_DIR"


RUN microdnf -y install gzip tar \
    && microdnf clean all

# Add a user with an explicit UID/GID and create necessary directories
RUN set -eux; \
    groupadd -r zookeeper --gid=1000; \
    useradd -r -g zookeeper --uid=1000 zookeeper; \
    mkdir -p "$ZOO_DATA_LOG_DIR" "$ZOO_DATA_DIR" "$ZOO_CONF_DIR" "$ZOO_LOG_DIR"; \
    chown zookeeper:zookeeper "$ZOO_DATA_LOG_DIR" "$ZOO_DATA_DIR" "$ZOO_CONF_DIR" "$ZOO_LOG_DIR"; \
    chown -R instana:0 "$ZOO_FIPS_DIR" \
    && chmod -R g=u "$ZOO_FIPS_DIR"

# Install required packges
RUN set -eux; \
    apt-get update; \
    DEBIAN_FRONTEND=noninteractive \
    apt-get install -y --no-install-recommends \
        ca-certificates \
        dirmngr \
        gosu \
        gnupg \
        netcat \
        wget; \
    rm -rf /var/lib/apt/lists/*; \
# Verify that gosu binary works
    gosu nobody true; \
    microdnf -y install gzip tar \
    && microdnf clean all

ARG GPG_KEY=DFF24FB8323ADAC90E3CF36F729E61230EA917E9
ARG SHORT_DISTRO_NAME=zookeeper-3.6.3
ARG DISTRO_NAME=apache-zookeeper-3.6.3-bin
ARG bcfips_sha256="5f4d12234904c61c6f12d74b6cf4b3b5d32a2c3375d67367735be000bdd979ab"
ARG artifact_base="https://artifact-rnd.instana.io/artifactory/instana-private/org/bouncycastle"


# Download Apache Zookeeper, verify its PGP signature, untar and clean up
RUN set -eux; \
    ddist() { \
        local f="$1"; shift; \
        local distFile="$1"; shift; \
        local success=; \
        local distUrl=; \
        for distUrl in \
            'https://www.apache.org/dyn/closer.cgi?action=download&filename=' \
            https://www-us.apache.org/dist/ \
            https://www.apache.org/dist/ \
            https://archive.apache.org/dist/ \
        ; do \
            if wget -q -O "$f" "$distUrl$distFile" && [ -s "$f" ]; then \
                success=1; \
                break; \
            fi; \
        done; \
        [ -n "$success" ]; \
    }; \
    ddist "$DISTRO_NAME.tar.gz" "zookeeper/$SHORT_DISTRO_NAME/$DISTRO_NAME.tar.gz"; \
    ddist "$DISTRO_NAME.tar.gz.asc" "zookeeper/$SHORT_DISTRO_NAME/$DISTRO_NAME.tar.gz.asc"; \
    export GNUPGHOME="$(mktemp -d)"; \
    gpg --keyserver hkps://keyserver.pgp.com --recv-key "$GPG_KEY" || \
    gpg --keyserver hkps://keyserver.ubuntu.com --recv-keys "$GPG_KEY" || \
    gpg --keyserver hkps://pgp.mit.edu --recv-keys "$GPG_KEY"; \
    gpg --batch --verify "$DISTRO_NAME.tar.gz.asc" "$DISTRO_NAME.tar.gz"; \
    tar -zxf "$DISTRO_NAME.tar.gz"; \
    mv "$DISTRO_NAME/conf/"* "$ZOO_CONF_DIR"; \
    rm -rf "$GNUPGHOME" "$DISTRO_NAME.tar.gz" "$DISTRO_NAME.tar.gz.asc"; \
    chown -R zookeeper:zookeeper "/$DISTRO_NAME"; \
    --mount=type=secret,id=artifactory_creds \
    export ARTIFACTORY_CREDS=$(< /run/secrets/artifactory_creds) \
    && curl -u "$ARTIFACTORY_CREDS" -fsSLo "/opt/instana/zookeeper/lib/bc-fips-$bcfips_version.jar" "$artifact_base/bc-fips/$bcfips_version/bc-fips-$bcfips_version.jar" \
    && sha256sum -c - <<< "$bcfips_sha256 /opt/instana/zookeeper/lib/bc-fips-$bcfips_version.jar" \
    && chown -R instana:0 /opt/instana/zookeeper


WORKDIR $DISTRO_NAME
VOLUME ["$ZOO_DATA_DIR", "$ZOO_DATA_LOG_DIR", "$ZOO_LOG_DIR"]

EXPOSE 2181 2888 3888 8080

ENV PATH=$PATH:/$DISTRO_NAME/bin \
    ZOOCFGDIR=$ZOO_CONF_DIR

COPY docker-entrypoint.sh /
HEALTHCHECK --start-period=1m --interval=30s --timeout=30s CMD /health
ENTRYPOINT ["/docker-entrypoint.sh"]
CMD ["zkServer.sh", "start-foreground"]
