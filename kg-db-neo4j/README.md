# Supported tags and respective `Dockerfile` links

* latest [(latest/Dockerfile)](https://github.com/ambiverse-nlu/dockerfiles/blob/master/kg-db-neo4j/Dockerfile)

# AmbiverseNLU Knowledge Graph Neo4j Database Dockerfile

This Dockerfile is an extension of the [neo4j:3.5.0](https://github.com/neo4j/docker-neo4j-publish/blob/bc0c1be414f5b671a681af8ac5dd8a5f83c02730/3.5.0/community/Dockerfile) official docker image. It creates a neo4j graph database version of [YAGO](http://yago-knowledge.org) with a name of the database specified by an environment variable, downloads the database dump and restores the database from the dump. With this image you are ready to use the [AmbiverseNLU Knowledge Graph](https://github.com/ambiverse-nlu/ambiverse-kg) service with neo4j graph database.

## Environment Variables
This image has several environment variables that need to be setup. Besides the environment variables from the [original image](https://hub.docker.com/r/_/neo4j/) that can be setup and are optional, some the following environment variables are mandatory. 

### DUMP_NAME

This environment variable is used to define a name for the database that is created when the image is first started. 

This environmental variable is also used to define the name of the database dump.  
The name must be chosen from the following list of dumps:

- **[yago_aida_en20180120_cs20180120_de20180120_es20180120_ru20180120_zh20180120](http://ambiversenlu-download.mpi-inf.mpg.de/neo4j/yago_aida_en20180120_cs20180120_de20180120_es20180120_ru20180120_zh20180120.tar.gz)** - full database dump of YAGO (Wikipedia dump from 20180120) nodes and relations  in six languages (czech, german, english, spanish, russian and chinese). Compressed, the file is `10GB` big, but once loaded in the `neo4j` container it will take around `100GB` of space.
- **[yago_aida_en20180120_de20180120_b38980492507620687f0729ddd2c43d2](http://ambiversenlu-download.mpi-inf.mpg.de/neo4j/yago_aida_en20180120_de20180120_b38980492507620687f0729ddd2c43d2.tar.gz)** - a small database dump of YAGO (Wikipedia dump from 20180120) nodes and relations in two languages (german and english). Compressed, the file is `450MB` big, but once loaded in the `neo4j` container it will take around `5GB` of space.

### NEO4J_dbms_active__database

This environment variable is from the original image, and sets the database defined from the `DUMP_NAME` as an active database. 

In the next section there is a command how to run neo4j with `docker run` and `docker-compose`. The environment variables used there are recommended and working for the size of the database dump.
Please note that if you change `NEO4J_AUTH` you have to adapt the `neo4j.properties` file in the [AmbiverseNLU KG](https://github.com/ambiverse-nlu/ambiverse-kg) accordingly, or link an external file to the KG container.

## Running the AmbiverseNLU container
To run the image from and connect it to directly with the [AmbiverseNLU KG](https://github.com/ambiverse-nlu/ambiverse-kg) service you can do it in the following two ways:

~~~~~~~~
docker run -d --restart=always --name kg-db-neo4j \
	-p 7474:7474 -p 7687:7687 \
	-e NEO4J_dbms_active__database=yago_aida_en20180120_cs20180120_de20180120_es20180120_ru20180120_zh20180120.db \
	-e NEO4J_dbms_memory_pagecache_size=8G \
	-e NEO4J_dbms_memory_heap_initial__size=8G \
	-e NEO4J_dbms_memory_heap_max__size=12G \
	-e NEO4J_dbms_connectors_default__listen__address=0.0.0.0 \
	-e NEO4J_dbms_security_procedures_unrestricted=apoc.* \
	-e NEO4J_AUTH=neo4j/neo4j_pass \
	-e DUMP_NAME=yago_aida_en20180120_cs20180120_de20180120_es20180120_ru20180120_zh20180120 \
	--ulimit=nofile=40000:40000 \
	-v $HOME/neo4j/data:/data \
	ambiverse/kg-db-neo4j
~~~~~~~~

&nbsp;

If you want to connect it to the AmbiverseNLU KG container, use the command below. This links the `kg-db-neo4j` container. The link name `kg-db-neo4j:db` is important, especially the part `:db`, since it is the host name defined in the `neo4j.properties` file in the [AmbiverseNLU KG](https://github.com/ambiverse-nlu/ambiverse-kg) project.
~~~~~~~~
docker run -d --restart=always --name ambiverse-kg \
 -p 8080:8080 \
 --link kg-db-neo4j:kg-db \
 ambiverse/ambiverse-kg
~~~~~~~~

&nbsp;

### ... or via `docker-stack deploy` or `docker-compose`
Example service-kg.yml for [AmbiverseNLU KG](https://github.com/ambiverse-nlu/ambiverse-kg):
~~~~~~~~
version: '3.6'

services:

  kg-db:
    image: ambiverse/kg-db-neo4j
    restart: always
    deploy:
      replicas: 1
    environment:
      DUMP_NAME: yago_aida_en20180120_cs20180120_de20180120_es20180120_ru20180120_zh20180120
      NEO4J_dbms_active__database: yago_aida_en20180120_cs20180120_de20180120_es20180120_ru20180120_zh20180120.db
      NEO4J_dbms_memory_pagecache_size: 8G
      NEO4J_dbms_memory_heap_initial__size: 8G
      NEO4J_dbms_memory_heap_max__size: 12G
      NEO4J_dbms_connectors_default__listen__address: 0.0.0.0
      NEO4J_dbms_security_procedures_unrestricted: apoc.*
      NEO4J_AUTH: neo4j/neo4j_pass
    ulimits:
      nofile:
        40000
    volumes:
      - type: volume
        source: dbdata
        target: /data
      - type: tmpfs
        target: /var/tmp/data
        tmpfs:
          size: 107374182400
    healthcheck:
      test: curl -sS http://127.0.0.1:7474/browser/ || exit 1
      interval: 1m
      timeout: 60s
      retries: 15
      start_period: 60m
    networks:
      kgnet:
        aliases:
          - kg-db

  kg:
    image: ambiverse/ambiverse-kg
    restart: always
    deploy:
      replicas: 1 # Increase the number of replicas if you want to scale horizontally
      resources:
        limits:
          #cpus: "1"
          memory: 16G
      restart_policy:
        condition: on-failure
    depends_on:
      - kg-db
    ports:
      - 8080:8080
    networks:
      - kgnet
    healthcheck:
      test: curl -sS http://127.0.0.1:8080/v2/knowledgegraph/entities/Q567 || exit 1
      interval: 1m
      timeout: 60s
      retries: 10
      start_period: 10s

volumes:
  dbdata:

networks:
  kgnet:
~~~~~~~~

Run `docker stack deploy -c service-kg.yml ambiverse-kg` (or `docker-compose -f service-kg.yml up`), wait for it to initialize completely.
