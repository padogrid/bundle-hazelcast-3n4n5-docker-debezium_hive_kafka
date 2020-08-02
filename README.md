# Debezium-Hive-Kafka Hazelcast Connector

This bundle integrates Hazelcast with Debezium and Apache Hive for ingesting initial data and CDC records from MySQL into a Hazelcast cluster via a Kafka sink connector included in the `padogrid` distribution. It supports inserts, updates and deletes.

## Installing Bundle

This bundle supports Hazelcast 3.12.x and 4.0.

```console
install_bundle -download bundle-hazelcast-3n4-docker-debezium_hive_kafka
```

:exclamation: If you are running this demo on WSL, make sure your workspace is on a shared folder. The Docker volume it creates will not be visible otherwise.

## Use Case

This use case ingests data changes made in the MySQL database into a Hazelcast cluster via Kafka connectors and also integrates Apache Hive for querying Kafka topics as external tables and views. It extends [the original Debezium-Kafka bundle](https://github.com/padogrid/bundle-hazelcast-3n4-docker-debezium_kafka) with Docker compose, Apache Hive, and  the North Wind mock data for `customers` and `orders` tables. It includes the MySQL source connector and the `hazelcast-addon` Debezium sink connectors.

![Debezium-Hive-Kafka Diagram](/images/debezium-hive-kafka.png)

## Required Software

- Docker
- Docker Compose
- Maven

## Optional Software

- jq

## Building Demo

We must first build the demo by running the `build_app` command as shown below. This command copies the Hazelcast and `hazelcast-addon-core` jar files to the Docker container mounted volume in the `padogrid` directory so that the Hazelcast Debezium Kafka connector can include them in its class path.

```console
cd_docker debezium_hive_kafka; cd bin_sh
./build_app
```

Upon successful build, the `padogrid` directory should have jar files similar to the following:

```console
cd_docker debezium_hive_kafka
tree padogrid
```

```console
padogrid/
├── etc
│   └── hazelcast-client.xml
├── lib
│   ├── hazelcast-addon-common-0.9.3-SNAPSHOT.jar
│   ├── hazelcast-addon-core-4-0.9.3-SNAPSHOT.jar
│   └── hazelcast-enterprise-all-4.0.1.jar
├── log
└── plugins
    └── hazelcast-addon-core-4-0.9.3-SNAPSHOT-tests.jar
```


## Creating Hazelcast Docker Containers

Let's create a Hazelcast cluster to run on Docker containers as follows.

```console
create_docker -cluster hazelcast -host host.docker.internal
cd_docker hazelcast
```

If you are running Docker Desktop, then the host name, `host.docker.internal`, is accessible from the containers as well as the host machine. You can run the `ping` command to check the host name.

```console
ping host.docker.internal
```

If `host.docker.internal` is not defined then you will need to use the host IP address that can be accessed from both the Docker containers and the host machine. Run `create_docker -?` or `man create_docker` to see the usage.

```console
create_docker -?
```

If you are using a host IP other than `host.docker.internal` then you must also make the change in the Debezium Hazelcast connector configuration file as follows.

```console
cd_docker debezium_hive_kafka
vi padogrid/etc/hazelcast-client.xml
```

Replace `host.docker.internal` in `hazelcast-client.xml` with your host IP address.

```xml
<hazelcast-client ...>
   ...
   <network>
      <cluster-members>
         <address>host.docker.internal:5701</address>
         <address>host.docker.internal:5702</address>
      </cluster-members>
   </network>
   ...
</hazelcast-client>
```

## Create `perf_test` app

Create `perf_test` for ingesting mock data into MySQL:

```bash
create_app -app perf_test -name perf_test_hive
```

Set the MySQL user name and password for `perf_test_hive`:

```bash
cd_app perf_test_hive
vi etc/hibernate.cfg-mysql.xml
```

Set user name and password as follows:

```xml
                <property name="connection.username">debezium</property>
                <property name="connection.password">dbz</property>
```

## Starting Docker Containers

### 1. Start Hazelcast

```bash
cd_docker hazelcast
docker-compose up
```

### 2. Start Debezium

Start Zookeeper, Kafka, MySQL, Kafka Connect, Apache Hive containers:

```bash
cd_docker debezium_hive_kafka
docker-compose up
```

Create the `nw` database and grant all privileges to the user `debezium`:

```bash
cd bin_sh
./create_nw_db
```

There are three (3) Kafka connectors that we need to register. The MySQL connector is provided by Debezium and the data connectors are part of the PadoGrid distribution. 

```bash
./register_connector_mysql
./register_connector_data_customers
./register_connector_data_orders
```

### 3. Ingest mock data into the `nw.customers` and `nw.orders` tables in MySQL

```bash
cd_app perf_test_hive; cd bin_sh
./test_group -run -db -prop ../etc/group-factory.properties
```

### 4. Run Hive Beeline CLI

```
cd_app perf_test_hive; cd bin_sh
./run_beeline
```

Create and query `customers` external table

```sql
-- Create customers external table
drop table if exists customers;
CREATE EXTERNAL TABLE customers
(payload struct <after:struct<customerId:string,address:string,city:string,companyName:string,contactName:string,contactTitle:string,country:string,fax:string,phone:string,postalCode:string,region:string>>)
STORED BY 'org.apache.hadoop.hive.kafka.KafkaStorageHandler'
TBLPROPERTIES
("kafka.topic" = "customers",
"kafka.bootstrap.servers"="kafka:9092"
 );

-- Query customers external table
select payload.after.customerId,payload.after.address,payload.after.city,payload.after.companyName,payload.after.contactName,payload.after.contactTitle,payload.after.country,payload.after.fax,payload.after.phone,payload.after.postalCode,payload.after.region,`__timestamp` from customers;

-- Query data consumed within the past 10 minutes
select payload.after.customerId,payload.after.address,payload.after.city,payload.after.companyName,payload.after.contactName,payload.after.contactTitle,payload.after.country,payload.after.fax,payload.after.phone,payload.after.postalCode,payload.after.region,`__timestamp` from customers
where `__timestamp` >  1000 * to_unix_timestamp(CURRENT_TIMESTAMP - interval '10' MINUTES);
```

Create and query `customers` external view

```sql
-- Define a view of data consumed within the past 15 minutes
drop view if exists customers_view;
CREATE VIEW customers_view AS SELECT  payload.after.customerId,payload.after.address,payload.after.city,payload.after.companyName,payload.after.contactName,payload.after.contactTitle,payload.after.country,payload.after.fax,payload.after.phone,payload.after.postalCode,payload.after.region,`__timestamp`
 ADDED FROM customers
 WHERE `__timestamp` >  1000 * to_unix_timestamp(CURRENT_TIMESTAMP - interval '15' MINUTES);

-- Query customers_view
select * from customers_view
```

### Watch topics

```bash
cd_docker debezium_hive_kafka; cd bin_sh
./watch_topic customers
./watch_topic orders
```

### Run MySQL CLI

```bash
cd_docker debezium_hive_kafka; cd bin_sh
./run_mysql_cli
```

### Check Kafka Connect

```console
# Check status
curl -Ss -H "Accept:application/json" localhost:8083/ | jq

# List registered connectors 
curl -Ss -H "Accept:application/json" localhost:8083/connectors/ | jq
```

The last command should display the connectors that we registered previously.

```console
[
  "customers-sink",
  "nw-connector",
  "orders-sink"
]
```

### View Map Contents

To view the map contents, run the `read_cache` command as follows:

```console
cd_app perf_test_hive
./read_cache nw/customers
./read_cache nw/orders
```

**Output:**

```console
```

### Desktop

You can also install the desktop app, browse and query the map contents. The build_app script configures and deploys all the necessary files for this demo.

```console
create_app -app desktop
cd_app desktop; bin_sh
./build_app
```

Run the desktop.

```console
cd bin_sh
./desktop
```

![Desktop Screenshot](/images/desktop-inventory-customers.png)

## Tearing Down

```console
# Shutdown Debezium containers
cd_docker debezium_hive_kafka
docker-compose down

# Shutdown Hazelcast containers
cd_docker hazelcast
docker-compose down

# Prune all stopped containers 
docker container prune
```
