---
layout: post
title: Run an Oracle Database on Docker in 5 minutes [ENG]
date: 2020-04-30 20:00:00 +0000
description: How to run an Oracle Database on Docker in 5 minutes, english version
img: docker-oracledb/container_oracle.png # Add image post (optional)
fig-caption: docker-oracledb/docker_oracle.png # Add figcaption (optional)
tags: [docker, oracle, database, 5min]
---

I really wanted to write a short article on how to install a complete Oracle Enterprise Database on your local machine. After some research I have found that the easiest and cleanest way is to use Docker. Using Docker we can take advantage of the many "images" on the Docker hub to experiment, test or play with many products in their original versions or if we want in the modified and extended versions by other developers.

This time my goal is simply to install a full local version of the Oracle Database in order to perform some tests. After the tests, I want to be free to stop the container and clean up everything.

### What is needed?
The first thing you need to do is to make sure you have a working version of Docker. If you haven't installed docker yet, do it now from: <a href="https://docs.docker.com/">docs.docker.com</a>

If you want to check that docker is working, open a shell and run the following command:
{% highlight shell %}
docker run hello-world
{% endhighlight %}
You should see the hello world image download and then the output related to the execution.

Next you will need to create a DockerHub account from <a href="https://hub.docker.com/">hub.docker.com</a>. At this point, you can search the desired image with the name "Oracle Database Enterprise Edition":

![docker-hub-oracledb]({{site.baseurl}}/assets/img/docker-oracledb/docker-hub.png)

As we can see, the image is created and managed directly by Oracle and Docker certifies it. We can also note that the image is based on Oracle Database Server 12.2.0.1 Enterprise Edition which will run on Oracle Linux 7. Another thing we can note is that it will be necessary to "checkout" the image and accept the defined terms from Oracle. Once this is done, we can take action.

### How you do it?
At this point, we must download the docker image, however to download this image we are required to login first:
{% highlight shell %}
docker login
{% endhighlight %}

then we download the image. Note that just because the database is an "Enterprise Edition" version on an Oracle Linux 7, the image size is quite bulky (4Gb for the basic version), I decided to use the slim image variant for this " 12.2.0.1-slim "which instead occupies only 2Gb. The slim variant does not support the following features: Analytics, Oracle R, Oracle Label Security, Oracle Text, Oracle Application Express and Oracle DataVault.

{% highlight shell %}
docker pull store/oracle/database-enterprise:12.2.0.1-slim
{% endhighlight %}

After downloading the image, we are one step away from finally being able to run the container of our database using the following command:

{% highlight shell %}
docker run -d -it --name oracle-db -p 1521:1521 store/oracle/database-enterprise:12.2.0.1-slim
{% endhighlight %}

Note that I have not specified any volumes so when I stop the instance I will lose all the data, for this first example it can also be fine. I also exposed port 1521 and did not specify any other parameters for which the database will have all the default setup.

The container will take some time to be running, so for this reason it will be important to understand if the initialization has finished. To do this we can consult the logs:
{% highlight shell %}
docker logs -f oracle-db
{% endhighlight %}

To conclude that the database is up and running, we can check the status of the container and verify that it is in "healty":
{% highlight shell %}
docker ps -f name=oracle-db
{% endhighlight %}

The container is running and we can finally interact with our database.

### Basic database setup
What we have now is an empty database so we need to create basic configurations. To do this, I recommend logging into sqlplus by typing the command:
{% highlight shell %}
docker exec -it oracle-db bash -c "source /home/oracle/.bashrc; sqlplus /nolog"

SQL*Plus: Release 12.2.0.1.0 Production on Sat Apr 18 17:06:54 2020

Copyright (c) 1982, 2016, Oracle.  All rights reserved.

SQL>
{% endhighlight %}

from here we are free to perform all the configurations we want for the database. Here I propose a basic version to define a user that I will call "myuser", assign the user the basic grants and unlimited tablespaces:

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

From this moment we can access the database through any client, for example Oracle SQL Developer:
![OracleSQLDeveloper]({{site.baseurl}}/assets/img/docker-oracledb/OracleSQLDeveloper.PNG)


#### Stop and clean the container

{% highlight shell %}
#stop the container
docker stop oracle-db
#remove the container
docker rm oracle-db
{% endhighlight %}
