# Debezium-Hive-Kafka Hazelcast Connector

This bundle integrates Hazelcast with Debezium and Apache Hive for ingesting initial data and CDC records from MySQL into a Hazelcast cluster via a Kafka sink connector included in the `padogrid` distribution. It supports inserts, updates and deletes.

## Installing Bundle

This bundle supports Hazelcast 3.12.x, 4.x, and 5.x.

```console
install_bundle -download bundle-hazelcast-3n4n5-docker-debezium_hive_kafka
```

:exclamation: If you are running this demo on WSL, make sure your workspace is on a shared folder. The Docker volume it creates will not be visible otherwise.

## Use Case

This use case ingests data changes made in the MySQL database into a Hazelcast cluster via Kafka connectors and also integrates Apache Hive for querying Kafka topics as external tables and views. It extends [the original Debezium-Kafka bundle](https://github.com/padogrid/bundle-hazelcast-3n4-docker-debezium_kafka) with Docker compose, Apache Hive, and  the North Wind mock data for `customers` and `orders` tables. It includes the MySQL source connector and the `hazelcast-addon` Debezium sink connectors.

![Debezium-Hive-Kafka Diagram](images/debezium-hive-kafka.jpg)

## Required Software

- Docker
- Docker Compose
- Maven 3.x
- PadoGrid 0.9.12-SNAPSHOT+ (10/18/2021) - for Hazelcast 5.x only

## Optional Software

- jq

## Building Demo

:pencil2: This bundle builds the demo enviroment based on the Hazelcast and Management versions in your workspace. Make sure your workspace has been configured with the desired versions before building the demo environment.

We must first build the demo by running the `build_app` command as shown below. This command copies the Hazelcast and `hazelcast-addon-core` jar files to the Docker container mounted volume in the `padogrid` directory so that the Hazelcast Debezium Kafka connector can include them in its class path. It also downloads the Hive JDBC driver jar and its dependencies in the `padogrid/lib/jdbc` directory.

```console
cd_docker debezium_hive_kafka/bin_sh
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
│   ├── hazelcast-addon-common-0.9.12-SNAPSHOT.jar
│   ├── hazelcast-addon-core-5-0.9.12-SNAPSHOT.jar
│   ├── hazelcast-enterprise-5.0.jar
│   └── jdbc
│       ├── commons-logging-1.2.jar
│       ├── curator-client-2.12.0.jar
│       ├── guava-19.0.jar
│       ├── hadoop-common-2.6.0.jar
│       ├── hive-common-3.1.2.jar
│       ├── hive-jdbc-3.1.2.jar
│       ├── hive-metastore-3.1.2.jar
│       ├── hive-serde-3.1.2.jar
│       ├── hive-service-3.1.2.jar
│       ├── hive-service-rpc-3.1.2.jar
│       ├── httpclient-4.5.2.jar
│       ├── httpcore-4.4.4.jar
│       ├── libthrift-0.9.3.jar
│       └── slf4j-api-1.7.10.jar
├── log
└── plugins
    └── hazelcast-addon-core-5-0.9.12-SNAPSHOT-tests.jar
```


## Creating Hazelcast Docker Containers

Let's create a Hazelcast cluster to run on Docker containers as follows.

```console
create_docker -product hazelcast -cluster hazelcast -host host.docker.internal
cd_docker hazelcast
```

If you named the cluster with a name other than `hazelcast`, then you need to update the `debezium_hive_kafka/docker-compose.yaml` file as follows.

```bash
cd_docker debezium_hive_kafka
vi docker-compose.yaml
```

Change the network name from `hazelcast_default` to your Hazelcast Docker cluster name. For example, if you named the cluster name as `my_new_hazelcast` then you would enter the folllowing. Make sure it ends with the `_default` postfix.

```yaml
networks:
  default:
    external:
      name: my_new_hazelcast_default
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

If you will be running the Desktop app then you also need to register the `org.hazelcast.demo.nw.data.PortableFactoryImpl` class in the Hazelcast cluster. The `Customer` and `Order` classes implement the `VersionedPortable` interface.

```bash
cd_docker hazelcast
vi padogrid/etc/hazelcast.xml
```

Add the following in the `hazelcast.xml` file.

```xml
            <portable-factory factory-id="1">
            org.hazelcast.demo.nw.data.PortableFactoryImpl
            </portable-factory>
```

## Creating `perf_test` app

Create and build `perf_test` for ingesting mock data into MySQL:

```bash
create_app -app perf_test -name perf_test_hive
cd_app perf_test_hive/bin_sh
./build_app
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

:exclamation: Wait till all the containers are up before executing the `init_all` script.

Execute `init_all` which performs the following:

- Create the `nw` database and grant all privileges to the user `debezium`:
- Copy the Kafka handler jar file to HDFS. It is required for executing queries with joins.

```bash
cd_docker debezium_hive_kafka/bin_sh
./init_all
```

There are three (3) Kafka connectors that we need to register. The MySQL connector is provided by Debezium and the data connectors are part of the PadoGrid distribution. 

```bash
cd_docker debezium_hive_kafka/bin_sh
./register_connector_mysql
./register_connector_data_customers
./register_connector_data_orders
```

### 3. Ingest mock data into the `nw.customers` and `nw.orders` tables in MySQL

:exclamation: Make sure to use the group-factory-**er**.properties file. This file creates entitiy relationships which we need for joining tables later.

```bash
cd_app perf_test_hive/bin_sh
./test_group -run -db -prop ../etc/group-factory-er.properties
```

### 4. Run Hive Beeline CLI

```
cd_docker debezium_hive_kafka/bin_sh
./run_beeline
```

Create and query `customers_payload` external table

```sql
-- Create customers external table
drop table if exists customers_payload;
CREATE EXTERNAL TABLE customers_payload
(payload string)
STORED BY 'org.apache.hadoop.hive.kafka.KafkaStorageHandler'
TBLPROPERTIES
("kafka.topic" = "customers",
"kafka.bootstrap.servers"="kafka:9092"
 );

-- Query customers_payload external table
select * from customers_payload;
```

**Output:**

```console
+----------------------------+----------------------------------------------------+--------------------------------+-----------------------------+--------------------------------+
| customers_payload.payload  |              customers_payload.__key               | customers_payload.__partition  | customers_payload.__offset  | customers_payload.__timestamp  |
+----------------------------+----------------------------------------------------+--------------------------------+-----------------------------+--------------------------------+
| {                          | {"schema":{"type":"struct","fields":[{"type":"string","optional":false,"field":"customerId"}],"optional":false,"name":"dbserver1.nw.customers.Key"},"payload":{"customerId":"k0000000612"}} | 0                              | 2900                        | 1596469641340                  |
| {                          | {"schema":{"type":"struct","fields":[{"type":"string","optional":false,"field":"customerId"}],"optional":false,"name":"dbserver1.nw.customers.Key"},"payload":{"customerId":"k0000000862"}} | 0                              | 2901                        | 1596469641340                  |
| {                          | {"schema":{"type":"struct","fields":[{"type":"string","optional":false,"field":"customerId"}],"optional":false,"name":"dbserver1.nw.customers.Key"},"payload":{"customerId":"k0000000987"}} | 0                              | 2902                        | 1596469641341                  |
| {                          | {"schema":{"type":"struct","fields":[{"type":"string","optional":false,"field":"customerId"}],"optional":false,"name":"dbserver1.nw.customers.Key"},"payload":{"customerId":"k0000000238"}} | 0                              | 2903                        | 1596469641341                  |
| {                          | {"schema":{"type":"struct","fields":[{"type":"string","optional":false,"field":"customerId"}],"optional":false,"name":"dbserver1.nw.customers.Key"},"payload":{"customerId":"k0000000113"}} | 0                              | 2904                        | 1596469641341                  |
| {                          | {"schema":{"type":"struct","fields":[{"type":"string","optional":false,"field":"customerId"}],"optional":false,"name":"dbserver1.nw.customers.Key"},"payload":{"customerId":"k0000000488"}} | 0                              | 2905                        | 1596469641341                  |
| {                          | {"schema":{"type":"struct","fields":[{"type":"string","optional":false,"field":"customerId"}],"optional":false,"name":"dbserver1.nw.customers.Key"},"payload":{"customerId":"k0000000363"}} | 0                              | 2906                        | 1596469641341                  |
...
```

Create and query `customers` external table


```sql
-- Create customers external table
drop table if exists customers;
CREATE EXTERNAL TABLE customers
(payload struct <after:struct<customerid:string,address:string,city:string,companyname:string,contactname:string,contacttitle:string,country:string,fax:string,phone:string,postalcode:string,region:string>>)
STORED BY 'org.apache.hadoop.hive.kafka.KafkaStorageHandler'
TBLPROPERTIES
("kafka.topic" = "customers",
"kafka.bootstrap.servers"="kafka:9092"
 );

-- Query customers external table
select payload.after.customerid,payload.after.address,payload.after.city,payload.after.companyname,payload.after.contactname,payload.after.contacttitle,payload.after.country,payload.after.fax,payload.after.phone,payload.after.postalCode,payload.after.region,`__partition`,`__timestamp` from customers;

-- Query data consumed within the past 10 minutes
select payload.after.customerId,payload.after.address,payload.after.city,payload.after.companyName,payload.after.contactName,payload.after.contactTitle,payload.after.country,payload.after.fax,payload.after.phone,payload.after.postalCode,payload.after.region,`__partition`,`__timestamp` from customers
where `__timestamp` >  1000 * to_unix_timestamp(CURRENT_TIMESTAMP - interval '10' MINUTES);
```

**Output:**

```console
+--------------+----------------------------------------------------+------------------------+---------------------------------------+----------------+---------------------------------------------+----------------------------------------------------+-----------------+------------------------+-------------+---------+--------------+----------------+
|  customerid  |                      address                       |          city          |              companyname              |  contactname   |                contacttitle                 |                      country                       |       fax       |         phone          | postalcode  | region  | __partition  |  __timestamp   |
+--------------+----------------------------------------------------+------------------------+---------------------------------------+----------------+---------------------------------------------+----------------------------------------------------+-----------------+------------------------+-------------+---------+--------------+----------------+
| k0000000612  | 50622 Boyer Rapids, Simonisberg, TN 62253          | Dickinsonhaven         | Pollich, Walker and Reichel          | Gibson         | Principal Marketing Officer                | Northern Mariana Islands                           | 714.873.0667    | (562) 943-2123 x838    | 64235-1513  | NE      | 0            | 1596469641340  |
| k0000000862  | 00081 Carlos Land, Aidaburgh, TN 12050             | Tremblayberg           | Miller, Bergstrom and Farrell        | Jenkins        | Corporate Manager                          | Svalbard & Jan Mayen Islands                       | 730-639-6453    | 546-194-4166 x9406     | 30365-8412  | NV      | 0            | 1596469641340  |
| k0000000987  | Suite 096 048 Ernser Crossing, Lake Chadville, WY 95944-1844 | Leschbury              | Sporer, Macejkovic and Bernier       | Anderson       | District Hospitality Executive             | Norway                                             | 801-027-1309    | 242.169.8662 x90534    | 66681       | MI      | 0            | 1596469641341  |
| k0000000238  | 2577 Sherri Row, Kohlerland, TX 64016              | Lianneland             | Baumbach LLC                         | Robel          | Legacy Facilitator                         | Sweden                                             | (148) 579-9627  | 360-235-2149 x07245    | 61811-7584  | AR      | 0            | 1596469641341  |
| k0000000113  | 098 Swift Camp, North Alana, IN 82409              | Heathhaven             | Moen and Sons                        | Turcotte       | Construction Supervisor                    | Reunion                                            | 1-053-688-2811  | (419) 464-3289         | 60529-1728  | PA      | 0            | 1596469641341  |
| k0000000488  | Suite 585 44094 Kertzmann Camp, Priceside, LA 04711 | Jerryland              | Kiehn-Hahn                           | Beahan         | Regional Strategist                        | Libyan Arab Jamahiriya                             | 1-186-101-3943  | 1-665-993-4497 x9829   | 61770-1776  | AZ      | 0            | 1596469641341  |
| k0000000363  | Apt. 162 970 Beahan Plains, Wintheiserton, FL 17306-9638 | West Melitaview        | Bayer-Mills                          | Herzog         | Product Accounting Officer                 | Honduras                                           | 1-703-981-5441  | 046-245-4210 x699      | 63782       | NM      | 0            | 1596469641341  |
| k0000000737  | 76469 Jennie Field, Connmouth, ND 75872            | New Petra              | Bogan, Jerde and Huel                | Wilderman      | Central Design Strategist                  | Albania                                            | (636) 618-0119  | 781.809.8438 x24523    | 52574       | MA      | 0            | 1596469641341  |
| k0000000613  | Apt. 882 311 Lola Orchard, Lake Omar, KY 01477-6493 | East Krystle           | Bosco LLC                            | Nolan          | Consulting Strategist                      | Bolivia                                            | 639.517.2600    | (952) 959-0903 x4037   | 54377       | VA      | 0            | 1596469641341  |
...
```

Create and query `customers` external view

```sql
-- Define a view of data consumed within the past 15 minutes
drop view if exists customers_view;
CREATE VIEW customers_view AS SELECT  payload.after.customerid,payload.after.address,payload.after.city,payload.after.companyname,payload.after.contactname,payload.after.contacttitle,payload.after.country,payload.after.fax,payload.after.phone,payload.after.postalcode,payload.after.region,`__partition`,`__timestamp`
 ADDED FROM customers
 WHERE `__timestamp` >  1000 * to_unix_timestamp(CURRENT_TIMESTAMP - interval '15' MINUTES);

-- Query customers_view
select * from customers_view;
```

**Output:**

```console
+----------------------------+----------------------------------------------------+------------------------+---------------------------------------+-----------------------------+---------------------------------------------+----------------------------------------------------+---------------------+------------------------+----------------------------+------------------------+-----------------------------+-----------------------+
| customers_view.customerid  |               customers_view.address               |  customers_view.city   |      customers_view.companyname       | customers_view.contactname  |         customers_view.contacttitle         |               customers_view.country               | customers_view.fax  |  customers_view.phone  | customers_view.postalcode  | customers_view.region  | customers_view.__partition  | customers_view.added  |
+----------------------------+----------------------------------------------------+------------------------+---------------------------------------+-----------------------------+---------------------------------------------+----------------------------------------------------+---------------------+------------------------+----------------------------+------------------------+-----------------------------+-----------------------+
| k0000000612                | 50622 Boyer Rapids, Simonisberg, TN 62253          | Dickinsonhaven         | Pollich, Walker and Reichel          | Gibson                      | Principal Marketing Officer                | Northern Mariana Islands                           | 714.873.0667        | (562) 943-2123 x838    | 64235-1513                 | NE                     | 0                           | 1596469641340         |
| k0000000862                | 00081 Carlos Land, Aidaburgh, TN 12050             | Tremblayberg           | Miller, Bergstrom and Farrell        | Jenkins                     | Corporate Manager                          | Svalbard & Jan Mayen Islands                       | 730-639-6453        | 546-194-4166 x9406     | 30365-8412                 | NV                     | 0                           | 1596469641340         |
| k0000000987                | Suite 096 048 Ernser Crossing, Lake Chadville, WY 95944-1844 | Leschbury              | Sporer, Macejkovic and Bernier       | Anderson                    | District Hospitality Executive             | Norway                                             | 801-027-1309        | 242.169.8662 x90534    | 66681                      | MI                     | 0                           | 1596469641341         |
| k0000000238                | 2577 Sherri Row, Kohlerland, TX 64016              | Lianneland             | Baumbach LLC                         | Robel                       | Legacy Facilitator                         | Sweden                                             | (148) 579-9627      | 360-235-2149 x07245    | 61811-7584                 | AR                     | 0                           | 1596469641341         |
| k0000000113                | 098 Swift Camp, North Alana, IN 82409              | Heathhaven             | Moen and Sons                        | Turcotte                    | Construction Supervisor                    | Reunion                                            | 1-053-688-2811      | (419) 464-3289         | 60529-1728                 | PA                     | 0                           | 1596469641341         |
| k0000000488                | Suite 585 44094 Kertzmann Camp, Priceside, LA 04711 | Jerryland              | Kiehn-Hahn                           | Beahan                      | Regional Strategist                        | Libyan Arab Jamahiriya                             | 1-186-101-3943      | 1-665-993-4497 x9829   | 61770-1776                 | AZ                     | 0                           | 1596469641341         |
...
```

Create and query `orders` external table

```sql
-- Create orders external table
drop table if exists orders;
CREATE EXTERNAL TABLE orders
(payload struct <after:struct<orderid:string,customerid:string,employeeid:string,freight:double,orderdate:bigint,requireddate:bigint,shipaddress:string,shipcity:string,shiptcountry:string,shipname:string,shippostcal:string,shipregion:string,shipvia:string,shippeddate:string>>)
STORED BY 'org.apache.hadoop.hive.kafka.KafkaStorageHandler'
TBLPROPERTIES
("kafka.topic" = "orders",
"kafka.bootstrap.servers"="kafka:9092"
 );

-- Query orders external table
select payload.after.orderid,payload.after.customerid,payload.after.employeeid,payload.after.freight,payload.after.orderdate,payload.after.requireddate,payload.after.shipaddress,payload.after.shipcity,payload.after.shiptcountry,payload.after.shipname,payload.after.shippostcal,payload.after.shipregion,payload.after.shipvia,payload.after.shippeddate,`__partition`,`__timestamp` from orders;

-- Query data consumed within the past 10 minutes
select payload.after.orderid,payload.after.customerid,payload.after.employeeid,payload.after.freight,payload.after.orderdate,payload.after.requireddate,payload.after.shipaddress,payload.after.shipcity,payload.after.shiptcountry,payload.after.shipname,payload.after.shippostcal,payload.after.shipregion,payload.after.shipvia,payload.after.shippeddate,`__partition`,`__timestamp` from orders
where `__timestamp` >  1000 * to_unix_timestamp(CURRENT_TIMESTAMP - interval '10' MINUTES);
```

**Output:**

```console
+--------------+--------------+--------------+-----------------------+----------------+----------------+----------------------------------------------------+-------------------------+---------------+-----------------------------------------+--------------+-------------+----------+----------------+--------------+----------------+
|   orderid    |  customerid  |  employeeid  |        freight        |   orderdate    |  requireddate  |                    shipaddress                     |        shipcity         | shiptcountry  |                shipname                 | shippostcal  | shipregion  | shipvia  |  shippeddate   | __partition  |  __timestamp   |
+--------------+--------------+--------------+-----------------------+----------------+----------------+----------------------------------------------------+-------------------------+---------------+-----------------------------------------+--------------+-------------+----------+----------------+--------------+----------------+
| k0000000732  | 526426+2257  | 096328-7565  | 49.75281954662483      | 1596432948000  | 1597441652000  | 88189 Kuhn Harbors, Lake Lowell, OR 24748          | New Ronny              | NULL          | Lowe, Renner and Sporer              | NULL         | TX          | 5        | 1596345855000  | 0            | 1596469652342  |
| k0000000611  | 419361-1507  | 964964-3342  | 101.46675311345439     | 1596229015000  | 1597206655000  | 2614 Dicki Forest, Marcelland, SD 50957-9374       | West Sanjuanita        | NULL          | Steuber, Stoltenberg and Roberts     | NULL         | SC          | 3        | 1596433994000  | 0            | 1596469652342  |
| k0000000107  | 984196-2058  | 852201-2339  | 128.41905675703595     | 1596080239000  | 1598164109000  | 45268 Stamm Views, Kassulkestad, FL 74590-2871     | Port Delbert           | NULL          | Senger-Gutmann                       | NULL         | TN          | 5        | 1596289310000  | 0            | 1596469652342  |
| k0000000990  | 477985+8129  | 717993-7500  | 98.00471176933468      | 1596464927000  | 1596723318000  | 050 Rafael Neck, Strackeside, LA 51000-4068        | South Criselda         | NULL          | Wiza, Schmeler and Daniel            | NULL         | OH          | 3        | 1596242501000  | 0            | 1596469652342  |
| k0000000491  | 859786-0524  | 782024-9205  | 20.145338636401444     | 1596151346000  | 1597210138000  | 3600 Schmitt Locks, Colettafort, KS 99443-5755     | Criseldaside           | NULL          | Mitchell-Luettgen                    | NULL         | OR          | 4        | 1596157697000  | 0            | 1596469652342  |
| k0000000238  | 442890+8548  | 785265+1717  | 159.2767378246903      | 1596351067000  | 1597763304000  | 055 Ortiz Track, New Chet, ID 65240                | Yelenamouth            | NULL          | Stracke, Ledner and Spencer          | NULL         | MO          | 5        | 1596391097000  | 0            | 1596469652342  |
| k0000000364  | 881110-0480  | 539429-7226  | 189.24358274264387     | 1595904644000  | 1597237824000  | 408 Murazik Bridge, Nelsonmouth, IN 73573          | West Nicky             | NULL          | Abbott, Walker and Thompson          | NULL         | WV          | 4        | 1596418615000  | 0            | 1596469652342  |
| k0000000991  | 271685+7676  | 348309-5741  | 136.71243143002636     | 1596019614000  | 1597387218000  | Suite 118 44901 Nathanael Motorway, North Diane, HI 83016-3989 | Deneseburgh            | NULL          | Parker-Mann                          | NULL         | PA          | 3        | 1596205754000  | 0            | 1596469652342  |
...
```

Create and query `orders` external view

```sql
-- Define a view of data consumed within the past 15 minutes
drop view if exists orders_view;
CREATE VIEW orders_view AS SELECT payload.after.orderid,payload.after.customerid,payload.after.employeeid,payload.after.freight,payload.after.orderdate,payload.after.requireddate,payload.after.shipaddress,payload.after.shipcity,payload.after.shiptcountry,payload.after.shipname,payload.after.shippostcal,payload.after.shipregion,payload.after.shipvia,payload.after.shippeddate,`__partition`,`__timestamp` 
 ADDED FROM orders
 WHERE `__timestamp` >  1000 * to_unix_timestamp(CURRENT_TIMESTAMP - interval '15' MINUTES);

-- Query orders
select * from orders_view;
```

**Output:**

```console
+----------------------+-------------------------+-------------------------+-----------------------+------------------------+---------------------------+----------------------------------------------------+-------------------------+---------------------------+-----------------------------------------+--------------------------+-------------------------+----------------------+--------------------------+--------------------------+--------------------+
| orders_view.orderid  | orders_view.customerid  | orders_view.employeeid  |  orders_view.freight  | orders_view.orderdate  | orders_view.requireddate  |              orders_view.shipaddress               |  orders_view.shipcity   | orders_view.shiptcountry  |          orders_view.shipname           | orders_view.shippostcal  | orders_view.shipregion  | orders_view.shipvia  | orders_view.shippeddate  | orders_view.__partition  | orders_view.added  |
+----------------------+-------------------------+-------------------------+-----------------------+------------------------+---------------------------+----------------------------------------------------+-------------------------+---------------------------+-----------------------------------------+--------------------------+-------------------------+----------------------+--------------------------+--------------------------+--------------------+
| k0000000732          | 526426+2257             | 096328-7565             | 49.75281954662483      | 1596432948000          | 1597441652000             | 88189 Kuhn Harbors, Lake Lowell, OR 24748          | New Ronny              | NULL                      | Lowe, Renner and Sporer              | NULL                     | TX                      | 5                    | 1596345855000            | 0
      | 1596469652342      |
| k0000000611          | 419361-1507             | 964964-3342             | 101.46675311345439     | 1596229015000          | 1597206655000             | 2614 Dicki Forest, Marcelland, SD 50957-9374       | West Sanjuanita        | NULL                      | Steuber, Stoltenberg and Roberts     | NULL                     | SC                      | 3                    | 1596433994000            | 0
      | 1596469652342      |
| k0000000107          | 984196-2058             | 852201-2339             | 128.41905675703595     | 1596080239000          | 1598164109000             | 45268 Stamm Views, Kassulkestad, FL 74590-2871     | Port Delbert           | NULL                      | Senger-Gutmann                       | NULL                     | TN                      | 5                    | 1596289310000            | 0
      | 1596469652342      |
| k0000000990          | 477985+8129             | 717993-7500             | 98.00471176933468      | 1596464927000          | 1596723318000             | 050 Rafael Neck, Strackeside, LA 51000-4068        | South Criselda         | NULL                      | Wiza, Schmeler and Daniel            | NULL                     | OH                      | 3                    | 1596242501000            | 0
      | 1596469652342      |
| k0000000491          | 859786-0524             | 782024-9205             | 20.145338636401444     | 1596151346000          | 1597210138000             | 3600 Schmitt Locks, Colettafort, KS 99443-5755     | Criseldaside           | NULL                      | Mitchell-Luettgen                    | NULL                     | OR                      | 4                    | 1596157697000            | 0
      | 1596469652342      |
| k0000000238          | 442890+8548             | 785265+1717             | 159.2767378246903      | 1596351067000          | 1597763304000             | 055 Ortiz Track, New Chet, ID 65240                | Yelenamouth            | NULL                      | Stracke, Ledner and Spencer          | NULL                     | MO                      | 5                    | 1596391097000            | 0
      | 1596469652342      |
| k0000000364          | 881110-0480             | 539429-7226             | 189.24358274264387     | 1595904644000          | 1597237824000             | 408 Murazik Bridge, Nelsonmouth, IN 73573          | West Nicky             | NULL                      | Abbott, Walker and Thompson          | NULL                     | WV                      | 4                    | 1596418615000            | 0
      | 1596469652342      |
...
```

Join `customers` and `orders`.

```sql
-- Join external tables
select c.payload.after.customerid,c.payload.after.address,o.payload.after.orderid,o.payload.after.customerid,o.payload.after.freight from customers c left outer join orders o on (c.payload.after.customerid=o.payload.after.customerid);
```

**Output:**

```console
+--------------+----------------------------------------------------+--------------+--------------+----------+
|  customerid  |                      address                       |   orderid    |  customerid  | freight  |
+--------------+----------------------------------------------------+--------------+--------------+----------+
| 000000-0072  | Apt. 591 63049 Nicolas Courts, Port Johnathan, CA 62844 | k0000000366  | 000000-0072  | 98.69    |
| 000000-0036  | Suite 556 2014 Robel Stream, Langoshmouth, VT 03041-5574 | k0000000186  | 000000-0036  | 60.13    |
| 000000-0036  | Suite 556 2014 Robel Stream, Langoshmouth, VT 03041-5574 | k0000000187  | 000000-0036  | 65.44    |
| 000000-0036  | Suite 556 2014 Robel Stream, Langoshmouth, VT 03041-5574 | k0000000188  | 000000-0036  | 9.46     |
| 000000-0036  | Suite 556 2014 Robel Stream, Langoshmouth, VT 03041-5574 | k0000000189  | 000000-0036  | 26.06    |
| 000000-0000  | 0653 Carter Knolls, Larondaland, IL 02519          | k0000000006  | 000000-0000  | 50.15    |
| 000000-0000  | 0653 Carter Knolls, Larondaland, IL 02519          | k0000000007  | 000000-0000  | 26.73    |
| 000000-0000  | 0653 Carter Knolls, Larondaland, IL 02519          | k0000000008  | 000000-0000  | 41.78    |
| 000000-0084  | Suite 073 484 Rhea Ferry, Ivoryshire, WI 47899-8001 | k0000000426  | 000000-0084  | 96.26    |
| 000000-0084  | Suite 073 484 Rhea Ferry, Ivoryshire, WI 47899-8001 | k0000000427  | 000000-0084  | 47.68    |
| 000000-0084  | Suite 073 484 Rhea Ferry, Ivoryshire, WI 47899-8001 | k0000000428  | 000000-0084  | 93.65    |
| 000000-0084  | Suite 073 484 Rhea Ferry, Ivoryshire, WI 47899-8001 | k0000000429  | 000000-0084  | 20.22    |
| 000000-0024  | Suite 752 571 Erdman Rapids, Evanside, WV 15189    | k0000000126  | 000000-0024  | 6.83     |
...
```

```sql
-- Join external views
select c.customerid,c.address,o.orderid,o.customerid,o.freight from customers_view c left outer join orders_view o on (c.customerid=o.customerid);
```

**Output:**

```console
+---------------+----------------------------------------------------+--------------+---------------+------------+
| c.customerid  |                     c.address                      |  o.orderid   | o.customerid  | o.freight  |
+---------------+----------------------------------------------------+--------------+---------------+------------+
| 000000-0072   | Apt. 591 63049 Nicolas Courts, Port Johnathan, CA 62844 | k0000000366  | 000000-0072   | 98.69      |
| 000000-0036   | Suite 556 2014 Robel Stream, Langoshmouth, VT 03041-5574 | k0000000186  | 000000-0036   | 60.13      |
| 000000-0036   | Suite 556 2014 Robel Stream, Langoshmouth, VT 03041-5574 | k0000000187  | 000000-0036   | 65.44      |
| 000000-0036   | Suite 556 2014 Robel Stream, Langoshmouth, VT 03041-5574 | k0000000188  | 000000-0036   | 9.46       |
| 000000-0036   | Suite 556 2014 Robel Stream, Langoshmouth, VT 03041-5574 | k0000000189  | 000000-0036   | 26.06      |
| 000000-0000   | 0653 Carter Knolls, Larondaland, IL 02519          | k0000000006  | 000000-0000   | 50.15      |
| 000000-0000   | 0653 Carter Knolls, Larondaland, IL 02519          | k0000000007  | 000000-0000   | 26.73      |
| 000000-0000   | 0653 Carter Knolls, Larondaland, IL 02519          | k0000000008  | 000000-0000   | 41.78      |
| 000000-0084   | Suite 073 484 Rhea Ferry, Ivoryshire, WI 47899-8001 | k0000000426  | 000000-0084   | 96.26      |
| 000000-0084   | Suite 073 484 Rhea Ferry, Ivoryshire, WI 47899-8001 | k0000000427  | 000000-0084   | 47.68      |
| 000000-0084   | Suite 073 484 Rhea Ferry, Ivoryshire, WI 47899-8001 | k0000000428  | 000000-0084   | 93.65      |
| 000000-0084   | Suite 073 484 Rhea Ferry, Ivoryshire, WI 47899-8001 | k0000000429  | 000000-0084   | 20.22      |
| 000000-0024   | Suite 752 571 Erdman Rapids, Evanside, WV 15189    | k0000000126  | 000000-0024   | 6.83       |
| 000000-0024   | Suite 752 571 Erdman Rapids, Evanside, WV 15189    | k0000000127  | 000000-0024   | 70.68      |
| 000000-0024   | Suite 752 571 Erdman Rapids, Evanside, WV 15189    | k0000000130  | 000000-0024   | 89.16      |
...
```

**Quit Beeline:**

```sql
!quit
```

### Watch topics

If you named the Hazlecast cluster other than `hazelcast`, then you need to update the `watch_topic` script. For example, if your cluster name is `my_new_hazelcast` as shown in the [Creating Hazelcast Docker Containers](#creating-hazelcast-docker-containers) section, then you need to update the `watch_topic` script as follows.

```bash
cd_docker debezium_hive_kafka/bin_sh
vi watch_topic
```

Update `watch_topic` as follows:

```bash
docker run --rm --tty --network my_new_hazelcast_default confluentinc/cp-kafkacat kafkacat -b kafka:9092 -t $1
```

Run `watch_topic` with a topic specified.

```bash
cd_docker debezium_hive_kafka/bin_sh
./watch_topic customers
./watch_topic orders
```

### Run MySQL CLI

```bash
cd_docker debezium_hive_kafka/bin_sh
./run_mysql_cli
```

Run join query (the same join query that fails to return results in BeeLine):

```sql
use nw;
select c.customerid,c.address,o.orderid,o.customerid,o.freight \
from customers c \
inner join orders o \
on (c.customerid=o.customerid) order by c.customerid,o.orderid limit 10;
```

**Output:**

```
+-------------+----------------------------------------------------+-------------+-------------+---------+
| customerid  | address                                            | orderid     | customerid  | freight |
+-------------+----------------------------------------------------+-------------+-------------+---------+
| 000000-0019 | 69781 Konopelski Union, South Cleo, NM 74341       | k0000000101 | 000000-0019 |   31.01 |
| 000000-0019 | 69781 Konopelski Union, South Cleo, NM 74341       | k0000000102 | 000000-0019 |   53.17 |
| 000000-0020 | Suite 405 0975 Howell Mission, Veumbury, TN 13258  | k0000000106 | 000000-0020 |   61.46 |
| 000000-0020 | Suite 405 0975 Howell Mission, Veumbury, TN 13258  | k0000000107 | 000000-0020 |    54.4 |
| 000000-0020 | Suite 405 0975 Howell Mission, Veumbury, TN 13258  | k0000000108 | 000000-0020 |   26.24 |
| 000000-0020 | Suite 405 0975 Howell Mission, Veumbury, TN 13258  | k0000000109 | 000000-0020 |    2.78 |
| 000000-0020 | Suite 405 0975 Howell Mission, Veumbury, TN 13258  | k0000000110 | 000000-0020 |   42.97 |
| 000000-0021 | Apt. 156 647 Batz Cove, East Burton, LA 62758-5125 | k0000000111 | 000000-0021 |   73.85 |
| 000000-0021 | Apt. 156 647 Batz Cove, East Burton, LA 62758-5125 | k0000000112 | 000000-0021 |   93.94 |
| 000000-0021 | Apt. 156 647 Batz Cove, East Burton, LA 62758-5125 | k0000000113 | 000000-0021 |   37.97 |
+-------------+----------------------------------------------------+-------------+-------------+---------+
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
  "nw-connector",
  "customers-sink",
  "orders-sink"
]
```

### View Map Contents

To view the map contents, run the `read_cache` command as follows:

```console
cd_app perf_test_hive/bin_sh
./read_cache nw/customers
./read_cache nw/orders
```

**Output:**

```console
...
        [address=Suite 579 23123 Drew Harbor, Coleburgh, OR 54795, city=Port Danica, companyName=Gulgowski-Weber, contactName=Howell, contactTitle=Forward Marketing Facilitator, country=Malaysia, customerId=000000-0878, fax=495.815.0654, phone=1-524-196-9729 x35639, postalCode=21468, region=ME]
        [address=74311 Hane Trace, South Devonstad, IA 99977, city=East Timmyburgh, companyName=Schulist-Heidenreich, contactName=Adams, contactTitle=Education Liaison, country=Djibouti, customerId=000000-0233, fax=074.842.7598, phone=959-770-3197 x7440, postalCode=68067-2632, region=NM]
        [address=22296 Toshia Hills, Lake Paulineport, CT 65036, city=North Lucius, companyName=Howe-Sporer, contactName=Bashirian, contactTitle=Human Construction Assistant, country=Madagascar, customerId=000000-0351, fax=(310) 746-2694, phone=284.623.1357 x04788, postalCode=73184, region=IA]
        [address=Apt. 531 878 Rosalia Common, South So, WV 38349, city=New Marniburgh, companyName=Hintz-Muller, contactName=Beier, contactTitle=Banking Representative, country=Tuvalu, customerId=000000-0641, fax=288-872-6542, phone=(849) 149-9890, postalCode=81995, region=MI]
...
```

### Desktop

You can also install the desktop app, browse and query the map contents. The build_app script configures and deploys all the necessary files for this demo.

```console
create_app -app desktop
cd_app desktop/bin_sh
./build_app
```

Run the desktop and login with your user ID and the default locator of localhost:5701. Password is not required.

```console
cd_app desktop; cd hazelcast-desktop_<version>/bin_sh
./desktop
```

![Desktop Screenshot](https://github.com/padogrid/bundle-hazelcast-3n4-docker-debezium_ksql_kafka/blob/master/images/desktop-nw-orders.jpg?raw=true)


### JDBC Browser

#### Hive

To browse Kafka stream data using Hive via JDBC, add all the jar files in the `padogrid/lib/jdbc` directory in the class path and configure your client with the following.

- JDBC URL: `jdbc:hive2://localhost:10000/default`
- Dirver Class Name: `org.apache.hive.jdbc.HiveDriver`
- User name and password are not required.

```bash
cd_docker debezium_hive_kafka
tree padogrid/lib/jdbc
```

**Output (JDBC jar files):**

```console
padogrid/lib/jdbc
├── commons-logging-1.2.jar
├── curator-client-2.12.0.jar
├── guava-19.0.jar
├── hadoop-common-2.6.0.jar
├── hive-common-3.1.2.jar
├── hive-jdbc-3.1.2.jar
├── hive-metastore-3.1.2.jar
├── hive-serde-3.1.2.jar
├── hive-service-3.1.2.jar
├── hive-service-rpc-3.1.2.jar
├── httpclient-4.5.2.jar
├── httpcore-4.4.4.jar
├── libthrift-0.9.3.jar
└── slf4j-api-1.7.10.jar
```

#### MySQL

You can configure MySQL JDBC as follows.

- JDBC URL: `jdbc:mysql://localhost:3306/nw`
- Driver Class Name: `com.mysql.cj.jdbc.Driver`
- User: `debezium`
- Password: `dbz`

The MySQL driver is located in the `perf_test_hive/lib` directory. You need only the `mysql-*.jar` file.

```bash
cd_app pert_test_hive
ls lib/mysql-*
```

**Output (JDBC jar file):**

```console
lib/mysql-connector-java-8.0.16.jar
```



SQuirreL SQL Client:

![SQuirreL SQL Client](images/hive-squirrel-client.jpg)

## Teardown

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

## References

1. Debizium-Kafka Hazelcast Connector, PadoGrid bundle, https://github.com/padogrid/bundle-hazelcast-3n4-docker-debezium_kafka
2. Debezium-KSQL-Kafka Hazelcast Connector, Padogrid bundle, https://github.com/padogrid/bundle-hazelcast-3n4-docker-debezium_ksql_kafka
3. Apache Hive, https://hive.apache.org
4. Apache Hive GitHub, https://github.com/apache/hive
