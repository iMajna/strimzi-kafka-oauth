FROM quay.io/strimzi/kafka:latest-kafka-2.8.0-arm64

COPY target/libs/* /opt/kafka/libs/oauth/
ENV CLASSPATH /opt/kafka/libs/oauth/*
