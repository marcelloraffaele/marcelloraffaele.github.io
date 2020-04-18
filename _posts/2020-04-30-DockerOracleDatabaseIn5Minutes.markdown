---
layout: post
title: Run an Oracle Database on Docker in 5 minutes [ENG]
date: 2020-04-30 20:00:00 +0000
description: How to run an Oracle Database on Docker in 5 minutes, english version
img: docker-oracledb/container_oracle.png # Add image post (optional)
fig-caption: docker-oracledb/docker_oracle.png # Add figcaption (optional)
tags: [docker, oracle, database, 5min]
---

#############################
{% highlight shell %}
docker run -d -it --name oracle-db-12c -p 9000:1521 store/oracle/database-enterprise:12.2.0.1-slim
{% endhighlight %}

#consulta i log
{% highlight shell %}
docker logs -f oracle-db-12c
{% endhighlight %}

#consulta lo stato, deve essere healty
{% highlight shell %}
docker ps -f name=oracle-db-12c
{% endhighlight %}

per consultare le porte esposte dal container
{% highlight shell %}
docker port oracle-db-12c
{% endhighlight %}

per accedere a sqlplus


{% highlight shell %}
docker exec -it oracle-db-12c bash -c "source /home/oracle/.bashrc; sqlplus /nolog"
{% endhighlight %}

bababa
{% highlight sql %}
connect sys/Oradoc_db1@ORCLCDB as sysdba

alter session set container=ORCLPDB1;

CREATE USER myuser IDENTIFIED BY myuser
	PROFILE default
	ACCOUNT UNLOCK;
GRANT CONNECT TO myuser;
GRANT RESOURCE TO myuser;
GRANT CREATE ANY VIEW TO myuser;
GRANT UNLIMITED TABLESPACE TO myuser;
{% endhighlight %}


{% highlight shell %}
docker stop oracle-db-12c
docker rm oracle-db-12c
{% endhighlight %}
