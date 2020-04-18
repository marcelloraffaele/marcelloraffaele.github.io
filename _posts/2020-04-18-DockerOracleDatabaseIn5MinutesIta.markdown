---
layout: post
title: Creare un Database Oracle in 5 minuti con Docker [ITA]
date: 2020-04-17 17:00:00 +0000
description: Creare un Database Oracle in 5 minuti con Docker, versione in italiano
img: docker-oracledb/container_oracle.png # Add image post (optional)
fig-caption: docker-oracledb/docker_oracle.png # Add figcaption (optional)
tags: [docker, oracle, database, 5min]
---

Da qualche giorno ormai volevo scrivere un breve articolo su come installare sulla propria macchina un Database Oracle Enterprise completo.
Dopo un po di ricerche ho scoperto che il modo più semplice è pulito è utilizzare Docker.
Utilizzando Docker possiamo sfruttare le numerosissime "immagini" presenti sul Docker hub per sperimentare, testare, giocare o semplicemente smanettare con tantissimi prodotti nelle loro versioni originali o se vogliamo nelle versioni modificate e estese da altri sviluppatori.

Questa volta il mio scopo era semplicemente quello di installare in locale una versione completa di Orale Database in modo da effettuare alcune prove. Finite le prove, voglio essere libero di arrestare il container e ripulire tutto.

### Cosa serve?
La prima cosa che bisogna fare è assicurarsi di avere una versione installata e funzionante di Docker.
Se non avete ancora installato docker, fatelo subito dal sito: <a href="https://docs.docker.com/">docs.docker.com</a>


Se volete verificare che docker sia funzionante, aprite una shell è lanciate il seguente comando:
{% highlight shell %}
docker run hello-world
{% endhighlight %}
Dovreste vedere scaricare l'immagine di hello world e successivamente l'output releativo all'esecuzione.

Successivamente sarà necessario creare un account per il DockerHub dal sito <a href="https://hub.docker.com/">hub.docker.com</a> effettua la registrazione.
A questo punto, sarà possibile cercare l'immagine desiderata con il nome "Oracle Database Enterprise Edition":

![docker-hub-oracledb]({{site.baseurl}}/assets/img/docker-oracledb/docker-hub.png)
Come possiamo notare, l'immagine è creata e gesatita direttamente da Oracle è Docker lo certifica. Possiamo notare anche che l'immagine è basata su Oracle Database Server 12.2.0.1 Enterprise Edition che verrà eseguito su Oracle Linux 7. Un' altra cosa che possiamo notare è che sarà necessario effettuare il "checkout" dell'immagine e accettare i termini definiti da Oracle. Effettuato ciò, possiamo passare all'azione.


### Come si fa?
A questo punto, dobbiamo scaricare l'immagine docker ed effettuare il pull, tuttavia per scaricare questa immagine ci è prima richiesto effettuare login:
{% highlight shell %}
docker login
{% endhighlight %}

successivamente scarichiamo l'immagine. Nota, proprio perchè il database è una versione "Enterprise Edition" su un Oracle Linux 7, la dimensione del'immagine risulta essere abbastanza ingombrante (4Gb per la versione base ), ho deciso di utilizzare per questo articola la variante slim dell'immagine "12.2.0.1-slim" che invece occupa solo 2Gb. La variante slim, non supporta le seguenti features: Analytics, Oracle R, Oracle Label Security, Oracle Text, Oracle Application Express and Oracle DataVault.
{% highlight shell %}
docker pull store/oracle/database-enterprise:12.2.0.1-slim
{% endhighlight %}

Terminato il download dell'immagine, siamo ad un passo dal poter finalmente eseguire il container contenete il nostro Database utilizzando il seguente comando:
{% highlight shell %}
docker run -d -it --name oracle-db -p 1521:1521 store/oracle/database-enterprise:12.2.0.1-slim
{% endhighlight %}
Nota: non ho specificato nessun volume per cui all'arresto dell'istanza perderò tutti i dati, per questo primo esempio può anche andare bene. Inoltre ho esposto la porta 1521 e non ho specificato nessun altro parametro per cui il database avrà tutto il setup di default.

Il container impiegherà un po di tempo per partire, per questo sarà importante comprendere se ha terminato l'inizializzazione e per fare ciò possiamo contultare i log:
{% highlight shell %}
docker logs -f oracle-db
{% endhighlight %}

Per concludere che il database è up e running, possiamo consultare lo stato del container e verificare che sia in "healty":
{% highlight shell %}
docker ps -f name=oracle-db
{% endhighlight %}


Il container è in esecuzione e possiamo interagire con il nostro database.

### Setup base del Database
Tuttavia quello che abbiamo è un database vuoto per cui abbiamo la necessità di creare delle configurazione base. Per fare questo, consiglio di accedere a sqlplus digitando il comando:
{% highlight shell %}
docker exec -it oracle-db bash -c "source /home/oracle/.bashrc; sqlplus /nolog"

SQL*Plus: Release 12.2.0.1.0 Production on Sat Apr 18 17:06:54 2020

Copyright (c) 1982, 2016, Oracle.  All rights reserved.

SQL>
{% endhighlight %}

da qui siamo liberi di eseguire tutte leconfigurazioni che desideriamo all'interno del database.
Io qui propongo una versione base per definire un utente che chiamerò "myuser", assegnare all'utente i grant base e infine tablespace illimitato:

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

Da questo momento in poi possiamo accedere al database mediante un qualsiasi client, per esempio Oracle SQL Developer:

![OracleSQLDeveloper]({{site.baseurl}}/assets/img/docker-oracledb/OracleSQLDeveloper.PNG)


#### Stop e pulizia del container

{% highlight shell %}
#stop del container
docker stop oracle-db
#eliminazione del container
docker rm oracle-db
{% endhighlight %}
