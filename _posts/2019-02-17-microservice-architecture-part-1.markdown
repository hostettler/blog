---
layout: post
comments: true
title: "Microservice Architecture - Part 1 (A running microservice architecture)"
date: 2019-02-17T22:51:20+01:00
---


## Introduction
This series of blog posts aims at helping students at the University of Geneva to develop their first application following micro-service principles. 
Besides explaining the concepts and implementation details of micro-service architecture, we will as well discuss software development practices such as software 
factories and innovative deployment options such as containers and container composition. All samples and a complete working application can be found [here on GitHub](https://github.com/hostettler/microservices.git)

The following diagram represents the end-state of our microservice architecture. From a business perspective, it delivers [RegTech](https://en.wikipedia.org/wiki/Regulatory_technology) services. 
More specifically, it manages counterparties and financial instruments. It valuates a portfolio and finally provides some regulatory reporting.
You do not need deep financial knowledge, sufficient is to say that:
- A counterparty is an individual or a company participating in a financial transaction. For more [details](https://www.investopedia.com/terms/c/counterparty.asp).
- A financial instrument  is an asset that can be traded such as stocks, loans, and the likes. For more [details](https://www.investopedia.com/terms/f/financialinstrument.asp).
- Portfolio valuation is the action of evaluating the net value of a set of assets. For more [details](https://www.investopedia.com/terms/a/assetvaluation.asp).
- Financial institutions must comply to a set of regulations such as delivering monthly reports to state their financial health.

{% include image.html url="/figures/micro-service-architecture.png" description="Network topology and high level component view of the micro-service architecture" %}

Besides, these "business" services, the architecture delivers a set of non-functional services such as:
- A Central logging mechanism to deal with the distributed nature of the architecture. It relies on a [Logspout](https://github.com/gliderlabs/logspout) companion container that sends the logs from all the containers to a concentrator called [Logstash](https://www.elastic.co) that in turn 
sends them to a database optimized for searching called [ElasticSearch](https://www.elastic.co). Finally, [Kibana](https://www.elastic.co) provides visualization and analysis of the logs.
- A Message broker to increase service decoupling and scalability. [Kafka](https://kafka.apache.org/) in this case.
- An API-Gateway that provides routing, load-balancing and SSO to the micro-services by integrating an identity manager called [Keycloack](https://www.keycloak.org/). Furthermore,  [Kong](https://konghq.com/kong/) delivers API-Gateway services (e.g., security, API composition and aggregation)
The API-Gateway also shields the user from knowing the ugly details of the network topology. Furthermore, it protects the backend by establishing a clear front vs back network separation,
 it exposes static resources and finally, it provides TLS termination.

From a technology perspective, Microservices are implemented using JEE 8  microservice and its microprofile. More specifically, [Thorntail](https://thorntail.io/) [3]. 
Furthermore, microservices are packaged as [Docker](https://www.docker.com/) [1][2] container using [Maven](https://maven.apache.org/) [4] as a build tool.

This chapter describes step by step how to compile and deploy the microservices themselves.
[Part 2](https://www.hostettler.net/2019/02/17/microservice-architecture-part-2.html) describes how to setup non-functional services such as SSO (Single Sign On), API concentration, and logging. Because of its
distributed nature, in 
a microservice architecture, the non-functional infrastructure is as important than the actual services.
[Part 3](https://www.hostettler.net/2019/03/04/microservice-architecture-part-3.html) dives deeper in what a microservice architecture actually is, its benefits and drawbacks, and some details on the related technologies.
Part 4 focuses on the software factory, putting everything together and testing the result. 
Finally, Part 5 does the autopsy of a microservice, detailing the associated design patterns.

# Pre-requisites

{% include info.html content="This series of blog post leverages a lot of different technologies. Please take the time to install everything properly. It will save time later on." %}

To execute the samples, you will need to install and to configure the following tools:
- a "reasonably" powerful computer with Linux (whatever recent distribution) or Windows (min. Windows 10) to support Docker. Mac is ok as well, but it requires some additional steps that will not be described here.
- a working Docker environment to deploy the services locally.
- a [JDK 11](https://jdk.java.net/11/) to compile and run the services
- a Git client for collaboration
- [NodeJS](https://nodejs.org/) and the [Angular tooling](https://angular.io/guide/quickstart)
- [Apache Maven](https://maven.apache.org/) for the automation.
- a bash interpreter (on Windows you can rely on Git bash that is usually installed with Git)


On top of that you need to have:
- An intermediate level in Java
- Some basic understanding of OS (including bash scripting) and networking (DNS, TCP, HTTP)
- a great deal of patience and coffee

{% include info.html content="We will start a lot of containers, please grant at least 6GB RAM and 6GB swap to your docker-machine" %}

## Getting the backend components to run
First things first, let's checkout the code and compile everything. Before you start complaining, 
yes, this section is tedious but we have to have the environment set up before diving into the wonderful world of microservices.
Let's start by cloning the code from [GitHub](https://github.com/hostettler/microservices.git). 

{% highlight bash %}
$ git clone https://github.com/hostettler/microservices.git
{% endhighlight %}
{% include console.html content="
Cloning into 'microservices'...
remote: Enumerating objects: 875, done.
remote: Counting objects: 100% (875/875), done.
remote: Compressing objects: 100% (596/596), done.
remote: Total 875 (delta 279), reused 787 (delta 203), pack-reused 0
Receiving objects: 100% (875/875), 3.68 MiB | 912.00 KiB/s, done.
Resolving deltas: 100% (279/279), done.
" %}

Let's check that Maven and Java are correctly installed.
{% highlight bash  %}
$ java -version
{% endhighlight %}
{% include console.html content="
openjdk version \"11.0.1\" 2018-10-16
OpenJDK Runtime Environment 18.9 (build 11.0.1+13)
OpenJDK 64-Bit Server VM 18.9 (build 11.0.1+13, mixed mode)
" %}

{% highlight bash  %}
$ mvn -v
{% endhighlight %}
{% include console.html content="
Apache Maven 3.5.4 (1edded0938998edf8bf061f1ceb3cfdeccf443fe; 2018-06-17T20:33:14+02:00)
Maven home: ...
Java version: 11.0.1, vendor: Oracle Corporation, runtime: ...
Default locale: en_US, platform encoding: Cp1252
OS name: \"windows 10\", version: \"10.0\", arch: \"amd64\", family: \"windows\"
" %}


The next step is to compile the project to produce the artifacts (i.e., binaries) that are required. To that end, 
we use Apache Maven. Maven is an opiniated build tool:
```
Opinionated Software is a software product that believes a certain way of approaching a business process is inherently
 better and provides software crafted around that approach.
```
Namely, following its opinion makes our life easier and requires less efforts. For more information and tutorials please refer to this [Maven Tutorial](https://maven.apache.org/guides/getting-started/maven-in-five-minutes.html). The output of the build process is a set of "JAR" files (i.e., JAVA library) that are stored in your local ``~/.m2`` repository for later use.

{% highlight bash %}
$ cd microservices/
$ mvn clean install
{% endhighlight %}
{% include console.html content="
[INFO] Scanning for projects...
[INFO] ------------------------------------------------------------------------
[INFO] Reactor Build Order:
[INFO]
[INFO] Parent Pom of the Pinfo Micro Services                             [pom]
[INFO] Counterparty Service                                               [war]
[INFO] Instrument Service                                                 [war]
[INFO] Valuation Service                                                  [war]
[INFO] Regulatory Reporting Service                                       [war]
[INFO] API Gateway Service                                                [war]
[INFO]
[INFO] -------------------< ch.unige:pinfo-micro-services >--------------------
[INFO] Building Parent Pom of the Pinfo Micro Services 0.2.0-SNAPSHOT     [1/6]
[INFO] --------------------------------[ pom ]---------------------------------
....
[INFO] ------------------------------------------------------------------------
[INFO] Reactor Summary:
[INFO]
[INFO] Parent Pom of the Pinfo Micro Services 0.2.0-SNAPSHOT SUCCESS [  3.216 s]
[INFO] Counterparty Service ............................... SUCCESS [ 42.866 s]
[INFO] Instrument Service ................................. SUCCESS [ 49.720 s]
[INFO] Valuation Service .................................. SUCCESS [ 32.623 s]
[INFO] Regulatory Reporting Service ....................... SUCCESS [ 23.048 s]
[INFO] API Gateway Service 0.2.0-SNAPSHOT ................. SUCCESS [ 22.796 s]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 02:55 min
[INFO] Finished at: 2019-02-20T18:09:09+01:00
[INFO] ------------------------------------------------------------------------
" %}

{% include success.html content="Congratulations, you compiled the microservices." %}

At this point, you have manage to compile all of the Java code and you have created maven artifacts for each microservice (Java Archives a.k.a. JARs). However,  as we continue, we will see in the next chapters that a micro-service architecture is much more than a bunch of micro-services. We will need a lot of additional 3rd party tools and services.
These additional services (e.g., logging, security) are usually provided as container images that runs on Docker. 
To be able to run the microservices along side these "3rd" party tools, we need to package the microservice as Docker images.

Simply put, Docker provides lightweight virtualization. It has a smaller footprint compare to the usual Virtual Machine approaches (Virtual Box, VM Ware).
The main difference is that the OS system layer is not replicated in each container but rather shared.

Docker containers run Docker images that are merely lightweight Linux systems with additional softwares. For more about [Docker](https://www.docker.com/)
Let's first check whether Docker is properly installed.

{% highlight bash %}
$ docker -v
{% endhighlight %}
{% include console.html content="
Docker version 18.09.2, build 6247962
" %}
{% highlight bash %}
$ docker run hello-world
{% endhighlight %}
{% include console.html content="
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
1b930d010525: Pull complete
Digest: sha256:2557e3c07ed1e38f26e389462d03ed943586f744621577a99efb77324b0fe535
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

...
" %}

Before we continue, let's have a look at a docker survival kit:
- ``docker ps`` displays all the running containers. 
- ``docker ps -a``displays all stopped container (not running but still using some resources). 

Based on that :
- we can kill all running containers by running ``docker kill $(docker ps -q)``
- remove all stopped containers by running ``docker rm $(docker ps -a -q)``
- finally ``docker system prune`` cleans up all dangling data.

So the Docker daemon is up and running. Let's create the Docker images for the microservices. This step will reuse the
JAR files created previously and package them along a Linux system so that every image can be run independently.

{% highlight bash %}
mvn install -Ppackage-docker-image
{% endhighlight %}
{% include console.html content="
INFO] ------------------------------------------------------------------------
[INFO] Reactor Summary:
[INFO]
[INFO] Parent Pom of the Pinfo Micro Services 0.2.0-SNAPSHOT SUCCESS [  6.114 s]
[INFO] Counterparty Service ............................... SUCCESS [ 59.134 s]
[INFO] Instrument Service ................................. SUCCESS [ 58.533 s]
[INFO] Valuation Service .................................. SUCCESS [ 42.806 s]
[INFO] Regulatory Reporting Service ....................... SUCCESS [ 33.979 s]
[INFO] API Gateway Service 0.2.0-SNAPSHOT ................. SUCCESS [ 14.134 s]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 03:35 min
[INFO] Finished at: 2019-02-20T18:58:34+01:00
[INFO] ------------------------------------------------------------------------
" %}

All the Docker images for the microservices have been created. Let's double check:
{% highlight bash %}
$ docker image ls | grep unige
{% endhighlight %}
{% include console.html content="
unige/regulatory-service       latest    5859668ecfb1        12 seconds ago       778MB
unige/valuation-service        latest    93516633b7b3        48 seconds ago       814MB
unige/instrument-service       latest    b1bded92050c        About a minute ago   814MB
unige/counterparty-service     latest    1789c8543673        2 minutes ago        780MB
unige/api-gateway              latest    b355613b0bbd        32 hours ago         371MB
" %}

{% include success.html content="You now have Docker images for your microservices. At this point, we have Docker images for the microservices and for the api-gateway." %}

Let's start a Docker container with the counterparty microservice and map the port ``28080`` of the container to the port ``10080`` of the host.
In principle, this will start a Linux OS and then start the microservice as the first process (PID 1). This container provides all the service, you would
expect from any Linux system such as network, security, and isolation.

{% highlight bash %}
$  docker run --name myCounterpartyService -p 10080:28080 unige/counterparty-service:latest
{% endhighlight %}
{% include console.html content="
2019-02-26 22:28:06,527 INFO  [org.jboss.as.server] (main) WFLYSRV0010: Deployed \"counterparty-service-0.2.0-SNAPSHOT.war\" (runtime-name : \"counterparty-service-0.2.0-SNAPSHOT.war\")
2019-02-26 22:28:06,569 INFO  [org.wildfly.swarm] (main) THORN99999: Thorntail is Ready
" %}
{% include success.html content="Open a browser and navigate to http://localhost:10080/counterparies It will display a long list of counterparties." %}
This demonstrates that a web services is listening on port ``10080`` of ``localhost``. More specifically, we started a container with the image of the counterparty microservice. The port ``28080`` is mapped to port ``10080`` so that we can test it.
Furthermore, we named the container `` myCounterpartyService``.

As it is a fully running Linux system, you can connect to the container to inspect it. In another console, we can run a ``docker ps`` command to list running containers.

{% highlight bash %}
docker ps
{% endhighlight %}
{% include console.html content='
CONTAINER ID        IMAGE                               COMMAND                  CREATED             STATUS              PORTS                     NAMES
dfb9acf07d79        unige/counterparty-service:latest   "/bin/sh -c \'java -D…"   42 seconds ago      Up 40 seconds       0.0.0.0:10080->8080/tcp   myCounterpartyService
' %}
As you can see,  there is one running container name ``myCounterpartyService`` that listen on port ``10080`` of ``localhost``.

Let's test it by connecting to ``http://localhost:10080/counterparties/724500J4K3Q60O9QLF45`` either by using a browser or the ``curl`` command line. ``counterparties`` is the context name of the service and ``724500J4K3Q60O9QLF45``is the id of one particular counterparty we want the details on.
{% highlight bash %}
curl -X GET http://localhost:10080/counterparties/724500J4K3Q60O9QLF45
{% endhighlight %}
{% include console.html content='
{"lei":"724500J4K3Q60O9QLF45","name":"Ton Smit Onroerend Goed B.V.","legalAddress":{"firstAddressLine":"Van Teylingenweg 126","city":"Kamerik","region":"","country":"NL","postalCode":"3471GG"},"registration":{"registrationAuthorityID":"RA000463","registrationAuthorityEntityID":"52431649","jurisdiction":"NL","legalFormCode":"54M6","category":"","registrationDate":1545264000000,"lastUpdated":1545264000000,"registrationStatus":"ISSUED","nextRenewalDate":1576800000000},"status":"ACTIVE"}
' %}

We can stop the service as follow:
{% highlight bash %}
docker stop myCounterpartyService
{% endhighlight %}

And check that nothing is running anymore:
{% highlight bash %}
docker ps
{% endhighlight %}
{% include console.html content="
CONTAINER ID        IMAGE                  COMMAND          CREATED     STATUS       PORTS          NAMES
" %}

So far we only ran one service, to run all the microservices (plus the message broker) we will compose the images by using ``docker-compose``.
``docker-compose`` is a way to script a series of complex Docker configuration to provide a coherent ecosystem.

{% highlight bash %}
cd docker-compose/
docker-compose -f docker-compose-microservices.yml up
{% endhighlight %}
{% include console.html content='
instrument-service    | 2019-02-28 07:55:54,602 INFO  [org.apache.kafka.clients.consumer.internals.AbstractCoordinator] (EE-ManagedExecutorService-default-Thread-1) [Consumer clientId=consumer-1, groupId=pinfo-microservices] Successfully joined group with generation 14
instrument-service    | 2019-02-28 07:55:54,606 INFO  [org.apache.kafka.clients.consumer.internals.ConsumerCoordinator] (EE-ManagedExecutorService-default-Thread-1) [Consumer clientId=consumer-1, groupId=pinfo-microservices] Setting newly assigned partitions [instrumentsReq-0]
valuation-service     | 2019-02-28 07:55:54,604 INFO  [org.apache.kafka.clients.consumer.internals.AbstractCoordinator] (EE-ManagedExecutorService-default-Thread-1) [Consumer clientId=consumer-1, groupId=pinfo-microservices] Successfully joined group with generation 14
valuation-service     | 2019-02-28 07:55:54,611 INFO  [org.apache.kafka.clients.consumer.internals.ConsumerCoordinator] (EE-ManagedExecutorService-default-Thread-1) [Consumer clientId=consumer-1, groupId=pinfo-microservices] Setting newly assigned partitions [instruments-0]
' %}

In another console, check the running containers
{% highlight bash %}
docker ps
{% endhighlight %}
{% include console.html content=' 
CONTAINER ID        IMAGE                             COMMAND                  CREATED              STATUS              PORTS                                               NAMES
f7af748fe9ae        unige/valuation-service:latest    "/bin/sh -c \'java -D…"   3 minutes ago       Up 3 minutes        0.0.0.0:12080->8080/tcp                             valuation-service
aea66ab35500        unige/instrument-service:latest   "/bin/sh -c \'java -D…"   3 minutes ago       Up 3 minutes        0.0.0.0:11080->8080/tcp                             instrument-service
2df3d6a8d6aa        confluentinc/cp-kafka:5.1.0       "/etc/confluent/dock…"   33 hours ago         Up 4 minutes        0.0.0.0:9092->9092/tcp                              kafka
f197de9c79fe        zookeeper:3.4.9                   "/docker-entrypoint.…"   33 hours ago         Up 4 minutes        2888/tcp, 0.0.0.0:2181->2181/tcp, 3888/tcp          zookeeper
' %}

Now we are ready to test the microservices. Let's check again that we can query counterparties.

{% highlight bash %}
curl -X GET http://localhost:10080/counterparties/724500J4K3Q60O9QLF45
{% endhighlight %}
{% include console.html content='
{"lei":"724500J4K3Q60O9QLF45","name":"Ton Smit Onroerend Goed B.V.","legalAddress":{"firstAddressLine":"Van Teylingenweg 126","city":"Kamerik","region":"","country":"NL","postalCode":"3471GG"},"registration":{"registrationAuthorityID":"RA000463","registrationAuthorityEntityID":"52431649","jurisdiction":"NL","legalFormCode":"54M6","category":"","registrationDate":1545264000000,"lastUpdated":1545264000000,"registrationStatus":"ISSUED","nextRenewalDate":1576800000000},"status":"ACTIVE"}
' %}

{% include warning.html content="In case you get an empty resultset, check that the database has been create successfully. If necessary remove the existing volume. Run the following command : docker volume rm --force microservices_pgdata-cpty " %}

Then let's get a specific instrument
{% highlight bash %}
curl -X GET http://localhost:11080/instrument/1
{% endhighlight %}
{% include console.html content='
{"id":1,"brokerLei":"254900LAW6SKNVPBBN21","counterpartyLei":"969500CHL179N00GX059","originalCurrency":"EUR","amountInOriginalCurrency":539926.20,"dealDate":-61630035780000,"valueDate":-61630035780000,"instrumentType":"B","isin":"BE7261065565","quantity":5445,"maturityDate":1577837340000}
' %}
Next, we will propagate all the instruments to the message broker for the valuation service to read them and compute the actual valuation.
{% highlight bash %}
curl -X POST http://localhost:11080/instrument/propagateAllInstruments
{% endhighlight %}
This is the actual result of the valuation of the portfolio.
{% highlight bash %}
curl -X GET http://localhost:12080/valuation?currency=USD
{% endhighlight %}
{% include console.html content='
{"breakdownByInstrumentType":{"STOCK":376127254.270,"LOAN":317483580.00,"BOND":468433784.120,"DEPOSIT":71056222.00,"WARRANT":4847202.120},"breakdownByCurrency":{"CHF":70073308.00,"SGD":66540948.00,"EUR":913601713.74,"GBP":102726326.00,"USD":85005746.77},"reportingCurrency":"USD","currentValue":1237948042.510,"percentile95":0.0,"percentile99":0.0}
' %}

{% include success.html content="Congrats, you just got all the microservies and the message broker running." %}

# Bibliography
- [1] N. Poulton, (2017) Docker Deep Dive
- [2] Turnbull, J. (2014). The Docker Book: Containerization is the new virtualization. 
- [3] Mauro Vocale, Luigi Fugaro (2018). Hands-On Cloud-Native Microservices with Jakarta EE
- [4] Raghuram Bharathan, (2015). Apache Maven Cookbook
