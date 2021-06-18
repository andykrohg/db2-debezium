# DB2 to Postgres Kafka Demo
This repo houses collateral to demonstrate a data streaming pipeline using the Debezium DB2 connector as a source and the Kafka Connect Postgres connector as a sink.

# How to deploy
1. Deploy a DB2 database to OpenShift. These manifests will generate a dummy database called `sample`.
   ```bash
   oc new-project db2
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
8. Create a `KafkaConnector` to start Debezium:
   ```bash
   oc apply -f openshift/kafka-connectors/db2-debezium-connector.yaml
   ```
9. Observe the creation of three new `KafkaTopics`:
   ```bash
   $ oc get kafkatopics -o name| grep public
   kafkatopic.kafka.strimzi.io/public.customers---8ca8dcfda9fa3ce95ed2659b19d107bbf03b0840
   kafkatopic.kafka.strimzi.io/public.orders---a03e6c4934252138bb668faa61013d62a0ab0b3a
   kafkatopic.kafka.strimzi.io/public.products---14d984dfd09e2a40f4d1929c00b6bb41eb11adfa
   ```
10. Create a JDBC Sink `KafkaConnector` to execute change events against Postgres:
   ```bash
   oc apply -f openshift/kafka-connectors/postgresql-sink-connector
   ```
11. Observe that the `CUSTOMERS`, `ORDERS`, and `PRODUCTS` tables have been created and populated with the base dataset in Postgres:
   ```bash
   oc rsh $(oc get pod -l name=postgresql -o name) psql sampledb -c \\dt
   ```
   ```
          List of relations
    Schema |   Name    | Type  |  Owner  
    --------+-----------+-------+---------
    public | CUSTOMERS | table | userOY8
    public | ORDERS    | table | userOY8
    public | PRODUCTS  | table | userOY8
    (3 rows)
   ```
   ```bash
   oc rsh $(oc get pod -l name=postgresql -o name) psql sampledb -c 'select * from "CUSTOMERS";'
   ```
   ```
    ID  | FIRST_NAME | LAST_NAME |         EMAIL         
    ------+------------+-----------+-----------------------
    1001 | Sally      | Thomas    | sally.thomas@acme.com
    1002 | George     | Bailey    | gbailey@foobar.com
    1003 | Edward     | Walker    | ed@walker.com
    1004 | Anne       | Kretchmar | annek@noanswer.org
   ```

12. In a new Terminal, add a new row to the `CUSTOMERS` table in the **DB2 Database**:
   ```bash
   oc rsh $(oc get pod -l app=db2 -o name) su - db2inst1
   ```
   ```bash
   db2 connect to sample
   ```
   ```bash
   db2 "INSERT INTO CUSTOMERS (FIRST_NAME, LAST_NAME, EMAIL) VALUES ('Andy', 'Krohg', 'akrohg@redhat.com')"
   ```

13. In the Postgres terminal (after a moment), observe that the new row appears in the `CUSTOMERS` table:
   ```bash
   oc rsh $(oc get pod -l name=postgresql -o name) psql sampledb -c 'select * from "CUSTOMERS";'
   ```

14. Try an `UPDATE` and `DELETE` too :)

