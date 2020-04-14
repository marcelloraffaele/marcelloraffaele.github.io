---
layout: post
title: Run an Oracle Database on Docker in 5 minutes
date: 2020-04-16 20:00:00 +0100
description: Simple first post
img: docker_oracle.png # Add image post (optional)
fig-caption: docker_oracle.png # Add figcaption (optional)
tags: [docker, oracle, database, 5min]
---

#############################
docker run -d -it --name oracle-db-12c -p 9000:1521 store/oracle/database-enterprise:12.2.0.1-slim

#consulta i log
docker logs -f oracle-db-12c

#consulta lo stato, deve essere healty
docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}' -f name=oracle-db-12c

#per consultare le porte esposte dal container
docker port oracle-db-12c

#per accedere a sqlplus
docker exec -it oracle-db-12c bash -c "source /home/oracle/.bashrc; sqlplus /nolog"
#sql
connect sys/Oradoc_db1@ORCLCDB as sysdba
alter session set container=ORCLPDB1;
--create user and assign grant
CREATE USER myuser IDENTIFIED BY myuser
    PROFILE default
    ACCOUNT UNLOCK;
GRANT CONNECT TO myuser;
GRANT RESOURCE TO myuser;
GRANT CREATE ANY VIEW TO myuser;
GRANT UNLIMITED TABLESPACE TO myuser;


##### STOP
docker ps -a;docker stop oracle-db-12c;docker rm oracle-db-12c;docker ps -a
