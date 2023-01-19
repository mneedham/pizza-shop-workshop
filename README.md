# Pizza Shop Workshop

## Pre Requisites

* Install kcat - https://docs.confluent.io/platform/current/app-development/kafkacat-usage.html
* Install jq - https://github.com/edenhill/kcat


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
kcat -C -b localhost:29092 -t orders
```


## Extra

If I get time:

* Update to use Swedish pizzas 
    * https://www.foodora.se/en/restaurant/o4ep/bella-pizza-by-foodle-city
    * https://www.foodora.se/en/restaurant/nq4i/4-corners-pizza-ostermalmshallen
