# Upstream source file for building is Dockerfile.ocp in
#       https://github.com/open-cluster-management/grafana
FROM registry-proxy.engineering.redhat.com/rh-osbs/rhacm2-dockerfile-drifter:latest
COPY $REMOTE_SOURCE $REMOTE_SOURCE_DIR
WORKDIR $REMOTE_SOURCE_DIR/app

COPY Dockerfile.ocp.cached Dockerfile.ocp.cached
RUN /dockerfile-drifter.sh Dockerfile.ocp Dockerfile.ocp.cached

#@follow_tag(registry-proxy.engineering.redhat.com/rh-osbs/openshift-golang-builder:rhel_8_1.17)
FROM registry-proxy.engineering.redhat.com/rh-osbs/openshift-golang-builder:v1.17.2-202111081951.el8.git.0ad85da AS builder

ENV COMPONENT_VERSION=2.4.1
ENV GOFLAGS=''

COPY $REMOTE_SOURCE $REMOTE_SOURCE_DIR
WORKDIR $REMOTE_SOURCE_DIR/app
RUN source $CACHITO_ENV_FILE && go run build.go build

# Need to copy the generated binaries to a non-platform specific location to handle
# s390x builds for example
RUN cp bin/linux-$(go env GOARCH)/grafana-server \
       bin/linux-$(go env GOARCH)/grafana-cli \
       /usr/bin/


#@follow_tag(registry.redhat.io/ubi8/ubi:latest)
FROM registry.redhat.io/ubi8/ubi:8.5-200

ARG GF_UID="472"
ARG GF_GID="472"

ENV PATH=/usr/share/grafana/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin \
    GF_PATHS_CONFIG="/etc/grafana/grafana.ini" \
    GF_PATHS_DATA="/var/lib/grafana" \
    GF_PATHS_HOME="/usr/share/grafana" \
    GF_PATHS_LOGS="/var/log/grafana" \
    GF_PATHS_PLUGINS="/var/lib/grafana/plugins" \
    GF_PATHS_PROVISIONING="/etc/grafana/provisioning"

WORKDIR $GF_PATHS_HOME

COPY --from=builder $REMOTE_SOURCE_DIR/app/conf ./conf

RUN mkdir -p "$GF_PATHS_HOME/.aws" && \
    # addgroup -S -g $GF_GID grafana && \
    # adduser -S -u $GF_UID -G grafana grafana && \
    # Note: the openshift base image does not include the addgroup and adduser commands so we have to use useradd/groupadd
    groupadd --system -g $GF_GID grafana && \
    useradd --system -u $GF_UID -g grafana grafana && \
    mkdir -p "$GF_PATHS_PROVISIONING/datasources" \
             "$GF_PATHS_PROVISIONING/dashboards" \
             "$GF_PATHS_PROVISIONING/notifiers" \
             "$GF_PATHS_LOGS" \
             "$GF_PATHS_PLUGINS" \
             "$GF_PATHS_DATA" && \
    cp "$GF_PATHS_HOME/conf/sample.ini" "$GF_PATHS_CONFIG" && \
    cp "$GF_PATHS_HOME/conf/ldap.toml" /etc/grafana/ldap.toml && \
    chown -R grafana:grafana "$GF_PATHS_DATA" "$GF_PATHS_HOME/.aws" "$GF_PATHS_LOGS" "$GF_PATHS_PLUGINS" "$GF_PATHS_PROVISIONING" && \
    chmod -R 777 "$GF_PATHS_DATA" "$GF_PATHS_HOME/.aws" "$GF_PATHS_LOGS" "$GF_PATHS_PLUGINS" "$GF_PATHS_PROVISIONING"

COPY --from=builder /usr/bin/grafana-server /usr/bin/grafana-cli ./bin/
COPY --from=builder $REMOTE_SOURCE_DIR/app/public ./public
COPY --from=builder $REMOTE_SOURCE_DIR/app/tools ./tools

EXPOSE 3000

COPY --from=builder $REMOTE_SOURCE_DIR/app/packaging/docker/run.sh /run.sh

USER grafana
ENTRYPOINT ["/run.sh"]

LABEL com.redhat.component="acm-grafana-container" \
      name="rhacm2/acm-grafana-rhel8" \
      version="v2.4.1" \
      summary="Grafana is an open-source, general purpose dashboard and graph composer" \
      io.openshift.expose-services="" \
      io.openshift.tags="openshift rhacm observability" \
      io.k8s.display-name="Grafana" \
      maintainer="['acm-component-maintainers@redhat.com']" \
      description="Grafana is an open-source, general purpose dashboard and graph composer; this image is built for ACM observability"

# 20210331
