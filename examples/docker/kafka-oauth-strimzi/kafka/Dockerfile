FROM quay.io/strimzi/kafka:latest-kafka-3.0.0-arm64

COPY libs/* /opt/kafka/libs/strimzi/
COPY config/* /opt/kafka/config/
COPY *.sh /opt/kafka/
COPY certificates/*.p12 /tmp/kafka/

USER root
RUN chmod +x /opt/kafka/*.sh
USER kafka

CMD ["/bin/bash", "/opt/kafka/start.sh"]