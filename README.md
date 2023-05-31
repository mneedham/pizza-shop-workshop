# Pizza Shop Workshop

This is a workshop where attendees learn how to build a Real-Time analytics dashboard for an imaginary pizza service. It acts as an introduction to the components of the Real-Time Analytics Stack. 

## Pre Requisites

* Install kcat - https://docs.confluent.io/platform/current/app-development/kafkacat-usage.html
* Install jq - https://github.com/edenhill/kcat
* Install psql - https://www.timescale.com/blog/how-to-install-psql-on-mac-ubuntu-debian-windows/

I'm using Pygmentize to view each of the files, but you can just view them in an editor.
You can install Pygmentize by running the following:

```bash
pip install Pygments
```

## Part 1

Set up Kafka and MySQL

```bash
pygmentize -O style=github-dark docker-compose-base.yml | less
```

```bash
docker-compose -f docker-compose-base.yml up
```

Look at the products

```bash
docker exec -it mysql mysql -u mysqluser -p
```

(Password is `mysqlpw`)

```sql
SELECT name, description, category, price 
FROM pizzashop.products 
LIMIT 10;
```

```sql
SELECT id, first_name, last_name, email, lat, lon
FROM pizzashop.users 
LIMIT 10;
```

Create an orders topic:

```bash
docker exec -it kafka \
  kafka-topics \
  --create \
  --bootstrap-server localhost:9092 \
  --topic orders \
  --partitions 5
```

Metadata for that topic:

```bash
kcat -L -b localhost:29092 -t orders
```

```bash
docker exec -it kafka \
  kafka-run-class \
  kafka.tools.GetOffsetShell \
  --broker-list localhost:9092 \
  --topic orders
```


## Part 2

Let's have a look at the Orders Service Simulator

```bash
pygmentize -O style=github-dark orders-service/multiseeder.py | less
```

And its accompanying Dockerfile and docker-compose files:

```bash
pygmentize -O style=github-dark orders-service/Dockerfile
pygmentize -O style=github-dark docker-compose-orders.yml | less
```

Now let's connect the simulator to our estate:

```bash
docker compose -f docker-compose-orders.yml up -d
```

```bash
kcat -C -b localhost:29092 -t orders | jq -c
kcat -C -b localhost:29092 -t orders -c1 | jq
```

## Part 3

Now let's get this data into Apache Pinot.

View the schema:

```bash
pygmentize -O style=github-dark pinot/config/orders/schema.json | less
```

View the table config:

```bash
pygmentize -O style=github-dark pinot/config/orders/table.json | less
```

Start Pinot:

```bash
docker compose -f docker-compose-pinot.yml up -d
```

Create the table:

```bash
docker run \
  -v $PWD/pinot/config:/config \
  --network pizza-shop \
  apachepinot/pinot:0.12.0-arm64 \
  AddTable \
  -schemaFile /config/orders/schema.json \
  -tableConfigFile /config/orders/table.json \
  -controllerHost pinot-controller \
  -exec
```

Navigate to the Pinot UI at http://localhost:9000/. 
Let's run some queries:

```sql
select ts, id, price, productsOrdered, totalQuantity, userId
from orders 
order by ts DESC
limit 10
```

```sql
select count(*), sum(price)
from orders 
WHERE ts > ago('PT1M')
order by ts DESC
limit 10
```

## Part 4

Now let's add a Streamlit dashboard populated by Pinot queries.

```bash
docker compose -f docker-compose-dashboard.yml up -d
```

Let's first look at the basic dashboard:

```bash
pygmentize -O style=github-dark streamlit/app_basic.py
```

Navigate to http://localhost:8501 to see the basic dashboard

And now the auto refreshing dashboard:

```bash
pygmentize -O style=github-dark streamlit/app.py
```

Navigate to http://localhost:8502 to see the auto-refreshing dashboard

## Part 5

Add Debezium to the estate

```bash
pygmentize -O style=github-dark docker-compose-debezium.yml | less
```

```bash
docker-compose -f docker-compose-debezium.yml up -d
```

Stream MySQL changes into Kafka

```bash
curl -X PUT -H  "Content-Type:application/json" http://localhost:8083/connectors/mysql/config \
    -d '{
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "database.hostname": "mysql",
    "database.port": 3306,
    "database.user": "debezium",
    "database.password": "dbz",
    "database.server.name": "mysql",
    "database.server.id": "223344",
    "database.allowPublicKeyRetrieval": true,
    "database.history.kafka.bootstrap.servers": "kafka:9092",
    "database.history.kafka.topic": "mysql-history",
    "database.include.list": "pizzashop",
    "time.precision.mode": "connect",
    "include.schema.changes": false
 }'
```

Products will be written to the `mysql.pizzashop.products` topic. 

```bash
kcat -C -b localhost:29092 -t mysql.pizzashop.products | jq
```

## Part 6

Adding Flink to the estate

```bash
pygmentize -O style=github-dark docker-compose-rwave.yml
```

```bash
docker compose -f docker-compose-rwave.yml up -d
```

Connect to Flink's CLI:

```bash
docker exec -it flink-sql-client /opt/sql-client/sql-client.sh
```

Create orders table:

```sql
CREATE TABLE Orders (
  `event_time` TIMESTAMP(3) METADATA FROM 'timestamp',
  `partition` BIGINT METADATA VIRTUAL,
  `offset` BIGINT METADATA VIRTUAL,
  `id` STRING,
  `userId` STRING,
  `price` DOUBLE,
  `items` ARRAY<ROW(`productId` STRING, `quantity` BIGINT, `price` DOUBLE)>,
  `deliveryLat` DOUBLE,
  `deliveryLon` DOUBLE,
  `createdAt` STRING
) WITH (
  'connector' = 'kafka',
  'topic' = 'orders',
  'properties.bootstrap.servers' = 'kafka:9092',
  'properties.group.id' = 'testGroup',
  'scan.startup.mode' = 'earliest-offset',
  'format' = 'json'
);
```

Query orders:

```sql
select id, userId, productId, quantity, t.price 
FROM Orders
CROSS JOIN UNNEST(items) AS t (productId, quantity, price);
```

Create products table:

```sql
CREATE TABLE Products (
  `event_time` TIMESTAMP(3) METADATA FROM 'timestamp',
  `partition` BIGINT METADATA VIRTUAL,
  `offset` BIGINT METADATA VIRTUAL,
  `id` STRING,
  `name` STRING,
  `description` STRING,
  `category` STRING,
  `price` DOUBLE,
  `image` STRING,
  `createdAt` STRING
) WITH (
  'connector' = 'kafka',
  'topic' = 'mysql.pizzashop.products',
  'properties.bootstrap.servers' = 'kafka:9092',
  'properties.group.id' = 'testGroup',
  'scan.startup.mode' = 'earliest-offset',
  'format' = 'debezium-json',
  'debezium-json.schema-include' = 'false'
);
```

Query products:

```sql
SELECT * 
FROM Products;
```

Join orders and products:

```sql
select 
  Orders.id AS orderId, 
  Orders.createdAt AS createdAt,
  MAP[
    'productId', CAST(orderItem.productId AS STRING),
    'quantity', CAST(orderItem.quantity AS STRING),
    'price', CAST(orderItem.price AS STRING)
   ] AS orderItem,
  MAP[
    'id', CAST(Products.id AS STRING),
    'name', CAST(Products.name AS STRING),
    'description', CAST(Products.description AS STRING),
    'category', CAST(Products.category AS STRING),
    'image', CAST(Products.image AS STRING),
    'price', CAST(Products.price AS STRING)
   ] AS product
FROM Orders
CROSS JOIN UNNEST(items) AS orderItem (productId, quantity, price)
JOIN Products ON Products.id = orderItem.productId;
```

Export as enriched order items:

```sql
CREATE TABLE EnrichedOrderItems (
  `orderId` STRING,
  `createdAt` STRING,
  `orderItem` MAP<STRING,STRING>,
  `product` MAP<STRING,STRING>,
   PRIMARY KEY (orderId) NOT ENFORCED
) WITH (
  'connector' = 'upsert-kafka',
  'topic' = 'enriched-order-items',
  'properties.bootstrap.servers' = 'kafka:9092',
  'properties.group.id' = 'testGroup',
  'value.format' = 'json',
  'key.format' = 'json'
);
```

Ingest joined data:

```sql
INSERT INTO EnrichedOrderItems
select 
  Orders.id AS orderId, 
  Orders.createdAt AS createdAt,
  MAP[
    'productId', CAST(orderItem.productId AS STRING),
    'quantity', CAST(orderItem.quantity AS STRING),
    'price', CAST(orderItem.price AS STRING)
   ] AS orderItem,
  MAP[
    'id', CAST(Products.id AS STRING),
    'name', CAST(Products.name AS STRING),
    'description', CAST(Products.description AS STRING),
    'category', CAST(Products.category AS STRING),
    'image', CAST(Products.image AS STRING),
    'price', CAST(Products.price AS STRING)
   ] AS product
FROM Orders
CROSS JOIN UNNEST(items) AS orderItem (productId, quantity, price)
JOIN Products ON Products.id = orderItem.productId;
```

We can then query the `enriched-order-items` stream:

```bash
kcat -C -b localhost:29092 -t enriched-order-items -c1 | jq
```

Query from the end of the stream:

```bash
kcat -C -b localhost:29092 -t enriched-order-items -o end | jq -c
```

## Part 7

Now let's add an enhanched dashboard, but first we'll add the `order_items_enriched` table:

```bash
pygmentize -O style=github-dark pinot/config/order_items_enriched/schema.json | less
pygmentize -O style=github-dark pinot/config/order_items_enriched/table.json | less
```

```bash
docker run \
  -v $PWD/pinot/config:/config \
  --network pizza-shop \
  apachepinot/pinot:0.11.0-arm64 \
  AddTable \
  -schemaFile /config/order_items_enriched/schema.json \
  -tableConfigFile /config/order_items_enriched/table.json \
  -controllerHost pinot-controller \
  -exec
```

And now the dashboard:

```bash
docker compose -f docker-compose-dashboard-enhanced.yml up -d
```

Navigate to http://localhost:8503

## Extra

If I get time:

* Update to use Swedish pizzas 
    * https://www.foodora.se/en/restaurant/o4ep/bella-pizza-by-foodle-city
    * https://www.foodora.se/en/restaurant/nq4i/4-corners-pizza-ostermalmshallen

* Nested JSON in Flink

```bash
CREATE TABLE Products2 (
  `payload` ROW<
    `after` ROW<
      `id` STRING,
      `name` STRING
    >
  >

) WITH (
  'connector' = 'kafka',
  'topic' = 'mysql.pizzashop.products',
  'properties.bootstrap.servers' = 'kafka:9092',
  'properties.group.id' = 'testGroup',
  'scan.startup.mode' = 'earliest-offset',
  'format' = 'json'
);
```

```sql
INSERT INTO EnrichedOrderItems
select Orders.id AS orderId, 
       Orders.createdAt AS createdAt,
       ROW(
        "orderItem.productId", "orderItem.productId",
        "orderItem.quantity", "orderItem.quantity",
        "orderItem.price", "orderItem.price"
       ) AS orderItem,
       ROW(
        'id', Products.id, 
        'name', Products.name, 
        'description', Products.description, 
        'category', Products.category,
        'image', Products.image,
        'price', Products.price
       ) AS product
FROM Orders
CROSS JOIN UNNEST(items) AS orderItem (productId, quantity, price)
JOIN Products ON Products.id = orderItem.productId;
```
