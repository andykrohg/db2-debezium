# DB2 to Postgres Kafka Demo
This repo houses collateral to demonstrate a data streaming pipeline using the Debezium DB2 connector as a source and the Kafka Connect Postgres connector as a sink.

# How to deploy
1. Deploy a DB2 database to OpenShift. These manifests will generate a dummy database called `sample`.
   ```bash
   oc apply -k openshift/db2/
   ```
2. Deploy a PostgreSQL Database using the ephemeral template in the Developer Catalog.
3. Install the Red Hat AMQ Streams Operator in the Web Console.
4. Build a db2connect image to use for your `KafkaConnect` cluster. Due to license requirements, be careful not to make this image publicly accessible.
   ```bash
   cd debezium-db2-init/db2connect
   podman login registry.redhat.io
   podman build -t quay.io/youruser/debezium-db2connect .
   podman push quay.io/youruser/debezium-db2connect
   ```
5. Replace **line 10** of `openshift/kafka/kafka-connect.yaml` to use your image. You'll most likely need to attach the appropriate pull-secret to the `my-connect-cluster-connect` service account.
6. Create `KafkaCluster` and `KafkaConnect` custom resources.
   ```bash
   oc apply -f openshift/kafka/kafka.yaml
   oc apply -f openshift/kafka/kafka-connect.yaml
   ```
7. Update line 12 in `openshift/kafka-connectors/postgresql-sink-connector` to use the correct username/password for postgres.
8. Create `KafkaConnector` resources:
   ```bash
   oc apply -f openshift/kafka-connectors/db2-debezium-connector.yaml
   oc apply -f openshift/kafka-connectors/posgresql-sink-connector.yaml
   ```