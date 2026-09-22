# One directory per plugin version. confluent-hub install writes every version
# into the same directory, so install each into a scratch directory and move it.
FROM confluentinc/cp-server-connect:8.3.2
ARG VERSIONS="10.9.7 10.9.8 10.9.9"
USER root
RUN set -eux; \
    for v in $VERSIONS; do \
      mkdir -p /tmp/hub-$v; \
      confluent-hub install --no-prompt --component-dir /tmp/hub-$v \
        --worker-configs /dev/null confluentinc/kafka-connect-jdbc:$v; \
      mv /tmp/hub-$v/confluentinc-kafka-connect-jdbc \
         /usr/share/confluent-hub-components/confluentinc-kafka-connect-jdbc-$v; \
      rm -rf /tmp/hub-$v; \
    done; \
    chown -R 1001:0 /usr/share/confluent-hub-components
USER 1001
