# Pizza Shop Workshop


https://www.foodora.se/en/restaurant/o4ep/bella-pizza-by-foodle-city
https://www.foodora.se/en/restaurant/nq4i/4-corners-pizza-ostermalmshallen


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

Orders Service Simulator

```bash
pygmentize orders-service/multiseeder.py | less
```

```bash
pygmentize orders-service/Dockerfile
```


```bash
docker compose -f docker-compose-orders.yml up -d
```

```bash
kcat -C -b localhost:29092 -t orders
```
