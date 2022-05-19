# About

Refreshing my memory about using Hive but in a local contained set of services using Docker and docker-compose. The idea is that we want to be able to set up a a hive cluster that uses a PostgreSQL metadata relational database to keep track of data across it's datanodes. We use PostgreSQL relational database instead of Apache Derby (the default) instead of HDFS for latency, demo and improved performance purposes.

## Inspo

I was literally inspired by this post: https://hshirodkar.medium.com/apache-hive-on-docker-4d7280ac6f8e

## Components and Services

**Name node**: central node in a hive cluster that keeps track of all data on data nodes using hive table. has environmental variables and volume data

**Data node**: built from the `hadoop-datanode` image on Dockerhub, this service contains the actual data distributed across our hive cluster. In this case, we have a cluster of 1. contains volume data mounted locally from `hdfs/datanode/` and relies on some environmental variable data from the `hadoop-hive.env` file as well as the `environment` instruction from the `docker-compose.yml`. 

**Hive Server**: This Hive server was built from the `Hive` image from Dockerhub. For metadata table store for your hive service, we use an hive image that specifies that use of Postgres as it's relational database with specifications/configs/env vars from the `hadoop-hive.env` file, the `environment` instruction in the `docker-compose.yml` and the volume data in the `employee` folder at the root directory. It depends on the `hive-metastore` service.

**Hive Metastore**: created using the hive-postgres image, it relies on `hive-metastore-postgres-sql` service before starting. This service has dependies from the environmental variables in the `hadoop-hive.env` file and the `environment` instruction in the `docker-compose.yml`

**Hive Metastore Postgresql**: based on the `hive-metastore-postgresql` image on Dockerhub, it acts as the relational database store for our metadata on our cluster. It depends on the `datanode` service container to be up.

## Walkthrough


### Starting it all up

In the root directory, run `docker-compose up -d`. This will start up all the necessary containers that will run : The `namenode`, `datanode`, `hive-server`, `hive-metastore`, and `hive-metastore-postgresql` services for our tutorial.  


### Ensuring that things were built and started correctly

We need to make sure that all of our services are upped correctly after the images have been built and the containers are started.

    docker stats


### Create the database and table

We can start with creating a database with a table using the HQL script (`employee_table.hql`) on the hive-server. 

We can start by shelling into the hive-server:

    docker exec -it

Run the HQL script that has the DDL to create the `testdb` database and the `employee` table using `hive`:

    cd ..
    cd employee
    hive -f employee_table.hql


### Initialize the table with data from CSV


While we are still in the hive-server container, we need to use `hadoop` to initialize the `employee` table with some records from our CSV file. While we are still in the `/employee` directory, from the above section, we can run:

    hadoop fs -put employee.csv hdfs://namenode:8020/user/hive/warehouse/testdb.db/employee


### Confirming that the new entries are in the database

Shell into the hadoop hive-server, that uses postgres as a metastore, and use the `hive` command to check if the new entries were created in the hive database

Run:

    docker exec -it hive-server
    hive
    show databases;
    use testdb;
    show tables;
    select * from employee;

A successful run of the last query should yield something like this:

    hive> select * from employee;
    OK
    1 Rudolf Bardin 30 cashier 100 New York 40000 5
    2 Rob Trask 22 driver 100 New York 50000 4
    3 Madie Nakamura 20 janitor 100 New York 30000 4
    4 Alesha Huntley 40 cashier 101 Los Angeles 40000 10
    5 Iva Moose 50 cashier 102 Phoenix 50000 20
    Time taken: 0.722 seconds, Fetched: 5 row(s)
