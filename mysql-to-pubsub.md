# Integrating Debezium with MySQL and Google Cloud Pub/Sub

This document provides guidance on configuring Debezium Server for capturing data changes from a MySQL database and publishing them to Google Cloud Pub/Sub. The setup includes Kubernetes deployment configuration and a dedicated `application.properties` file for Debezium.

## Kubernetes Deployment Configuration

The Debezium Server is deployed in Kubernetes, leveraging volumes for secrets, configuration, and persistent storage. The key elements of the Kubernetes deployment (`debezium-server.yaml`) are:

- **Volumes**:
  - `gcp-key`: A secret volume for Google Cloud credentials.
  - `config-volume`: A ConfigMap volume for the Debezium configuration.
  - `offsets-volume`: A PVC for storing offset data.
  - `schemahistory-volume`: A PVC for storing schema history.

- **Container Configuration**:
  - The container uses the `debezium/server` image.
  - Environment variable `GOOGLE_APPLICATION_CREDENTIALS` is set for GCP authentication.
  - Volume mounts for configuration, GCP credentials, offsets, and schema history.

### `debezium-server.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: debezium-server
  labels:
    app: debezium
spec:
  replicas: 1
  selector:
    matchLabels:
      app: debezium
  template:
    metadata:
      labels:
        app: debezium
    spec:
      volumes:
      - name: gcp-key
        secret:
          secretName: gcp-service-account
      - name: config-volume
        configMap:
          name: debezium-config
      - name: offsets-volume
        persistentVolumeClaim:
          claimName: debezium-offsets-pvc
      - name: schemahistory-volume
        persistentVolumeClaim:
          claimName: debezium-schemahistory-pvc
      containers:
      - name: debezium
        image: debezium/server
        env:
        - name: GOOGLE_APPLICATION_CREDENTIALS
          value: "/var/secrets/google/dev-test-key.json"
        volumeMounts:
        - name: config-volume
          mountPath: /debezium/conf
        - name: gcp-key
          mountPath: /var/secrets/google
        - name: offsets-volume
          mountPath: /tracker
        - name: schemahistory-volume
          mountPath: /schemahistory
```

### Persistent Volume Claims

Two PVCs are defined: `debezium-schemahistory-pvc` and `debezium-offsets-pvc`, each requesting 1Gi storage.

#### `debezium-schemahistory-pvc.yaml` and `debezium-offsets-pvc.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: debezium-schemahistory-pvc # or debezium-offsets-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

## Debezium Configuration (`application.properties`)

The `application.properties` file includes configurations for Debezium to connect to MySQL, manage schema history, and publish to Pub/Sub.

### Key Configurations:

- **Sink Type**: Configured to use Google Cloud Pub/Sub.
- **Schema History**: File-based, stored in `/schemahistory/FileDatabaseHistory.dat`.
- **Offset Storage**: Persistent volume at `/tracker/offsets.dat`.
- **MySQL Source Configuration**: Includes hostname, port, user, password, database name, and server id.
- **Pub/Sub Configuration**: The project id for Pub/Sub.

### `application.properties`:

```properties
# Sink, Schema History, Snapshot Mode, Engine Properties, Offset Storage
debezium.sink.type=pubsub
debezium.source.schema.history.internal=io.debezium.storage.file.history.FileSchemaHistory
debezium.source.schema.history.internal.file.filename=/schemahistory/FileDatabaseHistory.dat
debezium.source.snapshot.mode=initial
debezium.source.connector.class=io.debezium.connector.mysql.MySqlConnector
debezium.source.offset.storage.file.filename=/tracker/offsets.dat

# MySQL Source
debezium.source.database.hostname=10.102.251.226
debezium.source.database.port=3306
debezium.source.database.user=mysqluser
debezium.source.database.password=mysqlpw
debezium.source.database.dbname=inventory
debezium.source.database.server.name=debezium-test-db
debezium.source.database.server.id=123
debezium.source.database.include.list=inventory
debezium.source.table.include.list=inventory.customers

# Pub/Sub Configuration
debezium.source.topic.prefix=debezium-test-db
debezium.sink.pubsub.project.id=dev-test
```

### Ensuring Appropriate Database User Privileges

For Debezium to effectively capture data changes from a MySQL database, the database user configured for Debezium must be granted specific privileges. These privileges are essential for reading the binlog and capturing the schema changes.

#### Necessary Privileges

1. **RELOAD**: Needed for Debezium to fetch the latest schema.
2. **FLUSH_TABLES**: Allows Debezium to ensure a consistent state during snapshotting.
3. **REPLICATION SLAVE**: Required for reading binlog data.
4. **REPLICATION CLIENT**: Enables the user to get binary log information.
5. **SELECT**: Necessary for reading table data during snapshots.

#### Granting Privileges

Execute the following SQL statements on your MySQL server to grant these privileges:

```sql
GRANT RELOAD, FLUSH_TABLES ON *.* TO 'mysqluser'@'%';
GRANT REPLICATION SLAVE, REPLICATION CLIENT, SELECT ON *.* TO 'mysqluser'@'%';
FLUSH PRIVILEGES;
```

Replace `'mysqluser'@'%'` with the appropriate username and host for your Debezium setup.

#### Considerations

- **Security**: Granting such privileges, especially on `*.*`, can have security implications. It's crucial to ensure that the `mysqluser` is used exclusively for Debezium and is secured properly.
- **Specificity**: If possible, limit the privileges to the specific databases and tables that Debezium needs to access, rather than using `*.*`.
- **Environment**: These commands should be executed by a database administrator or someone with sufficient privileges on the MySQL server.

After executing these commands, the MySQL user will have the necessary rights to allow Debezium to capture data changes effectively.

## Key Configuration Elements

### Sink Type
- **Sink**: Google Cloud Pub/Sub
- **Property**: `debezium.sink.type=pubsub`

### Schema History
- **Type**: File-based schema history
- **Property**: `debezium.source.schema.history.internal=io.debezium.storage.file.history.FileSchemaHistory`
- **Storage Path**: `/schemahistory/FileDatabaseHistory.dat`

### Snapshot Mode
- **Default Mode**: `initial`
- **Recovery Mode**: `SCHEMA_ONLY_RECOVERY`
  - **Usage**: Employ `SCHEMA_ONLY_RECOVERY` mode only in case of lost history file or history topic in Kafka.

### Debezium Engine
- **Connector Class**: `io.debezium.connector.mysql.MySqlConnector`

### Offset Storage
- **Path**: `/tracker/offsets.dat`
  - **Note**: Persistent volume is recommended to maintain table offsets across instance lifecycles.

### MySQL Source Database Configuration
- **Hostname**: `debezium.source.database.hostname`
- **Port**: `debezium.source.database.port`
- **User**: `debezium.source.database.user`
- **Password**: `debezium.source.database.password`
- **Database Name**: `debezium.source.database.dbname`
- **Server Name**: `debezium.source.database.server.name`
- **Server ID**: `debezium.source.database.server.id`
- **Include List**: `debezium.source.database.include.list`
- **Table Include List**: `debezium.source.table.include.list`

### Topic Configuration
- **Topic Prefix**: `debezium.source.topic.prefix`

### Google Cloud Pub/Sub Configuration
- **Project ID**: `debezium.sink.pubsub.project.id`

## Steps to Operate Debezium with MySQL and Pub/Sub

1. **Prepare MySQL**: Ensure the MySQL user has required privileges (`REPLICATION SLAVE`, `REPLICATION CLIENT`) and the server is configured for row-based replication (`binlog_format=ROW`).

2. **Configure Debezium Server**: Use the properties outlined above. Adjust the `hostname`, `user`, `password`, and other MySQL-specific settings as per your environment.

3. **Schema History and Offsets Storage**: Ensure persistent storage for `/schemahistory/FileDatabaseHistory.dat` and `/tracker/offsets.dat`. This ensures continuity and consistency in schema and offset tracking.

4. **Snapshot Mode**: The default `initial` mode is suitable for first-time setup. In case of schema history loss, use `SCHEMA_ONLY_RECOVERY`.

5. **Google Cloud Pub/Sub Setup**: Ensure the Google Cloud project is correctly set up with Pub/Sub enabled, and the service account used has appropriate permissions.

6. **Start Debezium Server**: With the configuration in place, start the Debezium server. It should capture changes from the MySQL database and publish them to the configured Pub/Sub topic.

7. **Monitor and Troubleshoot**: Regularly monitor logs on the debezium server for any errors or issues, particularly during startup or schema changes in MySQL.

By following these configurations and steps, Debezium Server can effectively capture change data from MySQL and publish it to Google Cloud Pub/Sub, enabling real-time data streaming and integration use cases.