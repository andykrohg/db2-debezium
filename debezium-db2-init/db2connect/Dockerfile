FROM registry.redhat.io/amq7/amq-streams-kafka-26-rhel7:1.6.0

USER root:root
COPY ./plugins/ /opt/kafka/plugins/

# Deploy db2 client and drivers
RUN curl https://repo1.maven.org/maven2/com/ibm/db2/jcc/11.5.0.0/jcc-11.5.0.0.jar --output /opt/kafka/plugins/debezium-connector-db2/jcc-11.5.0.0.jar

USER 1001