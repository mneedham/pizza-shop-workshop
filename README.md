# Pizza Shop Workshop

## Pre Requisites

* Install kcat - https://docs.confluent.io/platform/current/app-development/kafkacat-usage.html
* Install jq - https://github.com/edenhill/kcat

I'm using Pygmentize to view each of the files, but you can just view them in an editor.
You can install Pygmentize by running the following:

```bash
pip install Pygments
```

## Part 1

Set up Kafka and MySQL

```bash
pygmentize docker-compose-base.yml | less
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
kcat -L -b localhost:29092 -t orders4
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
pygmentize orders-service/multiseeder.py | less
```

And its accompanying Dockerfile and docker-compose files:

```bash
pygmentize orders-service/Dockerfile
pygmentize docker-compose-orders.yml | less
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
pygmentize pinot/config/orders/schema.json | less
```

View the table config:

```bash
pygmentize pinot/config/orders/table.json | less
```

Add Pinot:

```bash
docker compose -f docker-compose-pinot.yml up -d
```

Create the table:

```bash
docker run \
  -v $PWD/pinot/config:/config \
  --network pizza-shop \
  apachepinot/pinot:0.11.0-arm64 \
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
pygmentize streamlit/app_basic.py
```

Navigate to http://localhost:8501 to see the basic dashboard

And now the auto refreshing dashboard:

```bash
pygmentize streamlit/app.py
```

Navigate to http://localhost:8501 to see the auto-refreshing dashboard


## Extra

If I get time:

* Update to use Swedish pizzas 
    * https://www.foodora.se/en/restaurant/o4ep/bella-pizza-by-foodle-city
    * https://www.foodora.se/en/restaurant/nq4i/4-corners-pizza-ostermalmshallen
