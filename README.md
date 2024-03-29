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
    "schema.history.internal.kafka.bootstrap.servers": "kafka:9092",    
    "schema.history.internal.kafka.topic": "mysql-schema-history",
    "database.include.list": "pizzashop",
    "time.precision.mode": "connect",
    "topic.prefix": "mysql",
    "include.schema.changes": false
}'
```

Products will be written to the `mysql.pizzashop.products` topic. 

```bash
kcat -C -b localhost:29092 -t mysql.pizzashop.products | jq
```

## Part 6

Adding RisingWave to the estate

```bash
pygmentize -O style=github-dark docker-compose-rwave.yml
```

```bash
docker compose -f docker-compose-rwave.yml up -d
```

Connect to the psql CLI:

```bash
psql -h localhost -p 4566 -d dev -U root
```

Create orders table:

```sql
CREATE SOURCE IF NOT EXISTS orders (
    id varchar,
    createdAt TIMESTAMP,
    userId integer,
    status varchar,
    price double,
    items STRUCT <
      productId varchar,
      quantity integer,
      price double
    >[]
)
WITH (
   connector='kafka',
   topic='orders',
   properties.bootstrap.server='kafka:9092',
   scan.startup.mode='earliest',
   scan.startup.timestamp_millis='140000000'
)
ROW FORMAT JSON;
```

Query orders:

```sql
WITH orderItems AS (
    select unnest(items) AS "orderItem",
           id AS "orderId", createdAt           
    FROM orders
)
select id, orders.createdat, orderItems.*
FROM orders
JOIN orderItems ON orderItems."orderId" = orders.id
LIMIT 10;
```

Create products table:

```sql
CREATE SOURCE IF NOT EXISTS products (
    id varchar,
    name varchar,
    description varchar,
    category varchar,
    price double,
    image varchar
)
WITH (
   connector='kafka',
   topic='products',
   properties.bootstrap.server='kafka:9092',
   scan.startup.mode='earliest',
   scan.startup.timestamp_millis='140000000'
)
ROW FORMAT JSON;
```

Query products:

```sql
SELECT * 
FROM Products;
```

Join orders and products:

```sql
WITH orderItems AS (
    select unnest(items) AS orderItem, 
           id AS "orderId", "createdAt           |
    FROM orders
)
SELECT "orderId", "createdAt",
       ((orderItem).productid, (orderItem).quantity, (orderItem).price)::
       STRUCT<productId varchar, quantity varchar, price varchar> AS "orderItem",
        (products.id, products.name, products.description, products.category, products.image, products.price)::
        STRUCT<id varchar, name varchar, description varchar, category varchar, image varchar, price varchar> AS product
FROM orderItems
JOIN products ON products.id = (orderItem).productId
LIMIT 10;
```

Export as materialized view:

```sql
CREATE MATERIALIZED VIEW orderItems_view AS
WITH orderItems AS (
    select unnest(items) AS orderItem, 
           id AS "orderId", createdAt AS "createdAt"
    FROM orders
)
SELECT "orderId", "createdAt",
       ((orderItem).productid, (orderItem).quantity, (orderItem).price)::
       STRUCT<productId varchar, quantity varchar, price varchar> AS "orderItem",
        (products.id, products.name, products.description, products.category, products.image, products.price)::
        STRUCT<id varchar, name varchar, description varchar, category varchar, image varchar, price varchar> AS product
FROM orderItems
JOIN products ON products.id = (orderItem).productId;
```

Create sink:

```sql
CREATE SINK enrichedOrderItems_sink FROM orderItems_view 
WITH (
   connector='kafka',
   type='append-only',
   properties.bootstrap.server='kafka:9092',
   topic='enriched-order-items'
);
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
  apachepinot/pinot:0.12.0-arm64 \
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