FROM quay.io/strimzi/kafka:latest-kafka-3.0.0-arm64

COPY target/libs/* /opt/kafka/libs/oauth/
ENV CLASSPATH /opt/kafka/libs/oauth/*
