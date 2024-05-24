<!--
  Copyright 2023 Datastrato Pvt Ltd.
  This software is licensed under the Apache License version 2.
-->

## Playground introduction

The playground is a complete Gravitino Docker runtime environment with `Hive`, `HDFS`, `Trino`, `MySQL`, `PostgreSQL`, `Jupter`, and a `Gravitino` server.

Depending on your network and computer, startup time may take 3-5 minutes. Once the playground environment has started, you can open [http://localhost:8090](http://localhost:8090) in a browser to access the Gravitino Web UI.

## Prerequisites

Install Git and Docker Compose.

## TCP ports used

The playground runs a number of services. The TCP ports used may clash with existing services you run, such as MySQL or Postgres.

| Docker container      | Ports used           |
|-----------------------|----------------------|
| playground-gravitino  | 8090 9001            |
| playground-hive       | 3307 9000 9083 50070 |
| playground-mysql      | 3306                 |
| playground-postgresql | 5342                 |
| playground-trino      | 8080                 |
| playground-jupyter    | 8888                 |

## Start playground

### Launch all components of playground
```shell
git clone git@github.com:datastrato/gravitino-playground.git
cd gravitino-playground
./launch-playground.sh
```

### Launch BigData components of playground
```shell
git clone git@github.com:datastrato/gravitino-playground.git
cd gravitino-playground
./launch-playground.sh bigdata
# equivalent to
./launch-playground.sh hive gravitino trino postgresql mysql spark
```

### Launch AI components of playground
```shell
git clone git@github.com:datastrato/gravitino-playground.git
cd gravitino-playground
./launch-playground.sh ai
# equivalent to
./launch-playground.sh hive gravitino mysql jupyter
```

### Launch special component or components of playground
```shell
git clone git@github.com:datastrato/gravitino-playground.git
cd gravitino-playground
./launch-playground.sh hive|gravitino|trino|postgresql|mysql|spark|jupyter
```

## Experiencing Gravitino with Trino SQL

### Using Trino CLI in Docker Container

1. Log in to the Gravitino playground Trino Docker container using the following command:

```shell
docker exec -it playground-trino bash
````

2. Open the Trino CLI in the container.

```shell
trino@container_id:/$ trino
```

### Using Jupiter Notebook

1. Open the Jupyter Notebook in the browser at [http://localhost:8888](http://localhost:8888).

2. Open the `gravitino-trino-example.ipynb` notebook.

3. Start the notebook and run the cells.

## Example

### Simple queries

You can use simple queries to test in the Trino CLI.

```SQL
SHOW CATALOGS;

CREATE SCHEMA catalog_hive.company
  WITH (location = 'hdfs://hive:9000/user/hive/warehouse/company.db');

SHOW CREATE SCHEMA catalog_hive.company;

CREATE TABLE catalog_hive.company.employees
(
  name varchar,
  salary decimal(10,2)
)
WITH (
  format = 'TEXTFILE'
);

INSERT INTO catalog_hive.company.employees (name, salary) VALUES ('Sam Evans', 55000);

SELECT * FROM catalog_hive.company.employees;

SHOW SCHEMAS from catalog_hive;

DESCRIBE catalog_hive.company.employees;

SHOW TABLES from catalog_hive.company;
```

### Cross-catalog queries

In a company, there may be different departments using different data stacks. In this example, the HR department uses Apache Hive to store its data and the sales department uses PostgreSQL. You can run some interesting queries by joining the two departments' data together with Gravitino.

To know which employee has the largest sales amount, run this SQL:

```SQL
SELECT given_name, family_name, job_title, sum(total_amount) AS total_sales
FROM catalog_hive.sales.sales as s,
  catalog_postgres.hr.employees AS e
where s.employee_id = e.employee_id
GROUP BY given_name, family_name, job_title
ORDER BY total_sales DESC
LIMIT 1;
```

To know the top customers who bought the most by state, run this SQL:

```SQL
SELECT customer_name, location, SUM(total_amount) AS total_spent
FROM catalog_hive.sales.sales AS s,
  catalog_hive.sales.stores AS l,
  catalog_hive.sales.customers AS c
WHERE s.store_id = l.store_id AND s.customer_id = c.customer_id
GROUP BY location, customer_name
ORDER BY location, SUM(total_amount) DESC;
```

To know the employee's average performance rating and total sales, run this SQL:

```SQL
SELECT e.employee_id, given_name, family_name, AVG(rating) AS average_rating, SUM(total_amount) AS total_sales
FROM catalog_postgres.hr.employees AS e,
  catalog_postgres.hr.employee_performance AS p,
  catalog_hive.sales.sales AS s
WHERE e.employee_id = p.employee_id AND p.employee_id = s.employee_id
GROUP BY e.employee_id,  given_name, family_name;
```

### Using Iceberg REST service

If you want to migrate your business from Hive to Iceberg. Some tables will use Hive, and the other tables will use Iceberg.
Gravitino provides an Iceberg REST catalog service, too. You can use Spark to access REST catalog to write the table data.
Then, you can use Trino to read the data from the Hive table joining the Iceberg table.

`spark-defaults.conf` is as follows (It's already configured in the playground):

```text
spark.sql.extensions org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions
spark.sql.catalog.catalog_iceberg org.apache.iceberg.spark.SparkCatalog
spark.sql.catalog.catalog_iceberg.type rest
spark.sql.catalog.catalog_iceberg.uri http://gravitino:9001/iceberg/
spark.locality.wait.node 0
```

1. Login Spark container and execute the steps.

```shell
docker exec -it playground-spark bash
```

```shell
spark@container_id:/$ cd /opt/spark && /bin/bash bin/spark-sql 
```

```SQL
use catalog_iceberg;
create database sales;
use sales;
create table customers (customer_id int, customer_name varchar(100), customer_email varchar(100));
describe extended customers;    
insert into customers (customer_id, customer_name, customer_email) values (11,'Rory Brown','rory@123.com');
insert into customers (customer_id, customer_name, customer_email) values (12,'Jerry Washington','jerry@dt.com');
```

2. Login Trino container and execute the steps.
You can get all the customers from both the Hive and Iceberg table.

```shell
docker exec -it playground-trino bash
```

```shell
trino@container_id:/$ trino  
```

```SQL
select * from catalog_hive.sales.customers
union
select * from catalog_iceberg.sales.customers;
```
