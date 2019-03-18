---
layout: post
comments: true
title: "Microservice Architecture - Part 2 (SSO, Logging, and all that)"
date: 2019-02-17T22:51:20+01:00
published: true
---

In [part 1](https://www.hostettler.net/2019/02/17/microservice-architecture-part-1.html), we discussed how to compile and deploy the microservices. Remember that the microservices themselves are only a part of the microservice architecture. 
By its very nature, microservice architecture are distributed and that comes with a lot of benefits and some constraints.
One of these constraint is that all the non-functional features such as security, logging, testability and so on, have to take distribution into account.
Think of the microservice architecture as a city, where the microservice are people working in the city. In the city, you also need policemen, firefighters, teachers, healthcare providers to keep it up and running.
The higher the number of people working in the private sector (a.k.a., microservices), the higher the need for non-operational people (a.k.a., utilities)

## Compile the UI
This sample microservice architecture does not focus much on the UI. It mainly serves the purpose of showing how to integrate
it with the rest of the architecture. We will not dive into details. Sufficient is to say, that the example was built with [Angular 7.0](https://angular.io/)
and the [ngx-admin dashboard](https://github.com/akveo/ngx-admin).
In development, UI is compiled by  and [npm](https://www.npmjs.com/) running on top of [nodejs](https://nodejs.org/en/).

{% highlight bash %}
$ cd web-ui
$ node --version
{% endhighlight %}
{% include console.html content="
10.15.0
" %}   
{% highlight bash %}
$ npm --version
{% endhighlight %}
{% include console.html content="
6.5.0
" %}   
{% highlight bash %}
$ npm install
{% endhighlight %}
{% include console.html content="
...
audited 31887 packages in 68.922s
" %}   

{% highlight bash %}
$ npm run-script build
{% endhighlight %}
{% include console.html content="
69% building modules 1280/1296 modules 16 active ...components\footer\footer.component.scssDEPRECATION WARNING on line 1, column 8 of 
Including .css files with @import is non-standard behaviour which will be removed in future versions of LibSass.
Use a custom importer to maintain this behaviour. Check your implementations documentation on how to create a custom importer.

Date: 2019-02-21T09:13:40.912Z
Hash: e3a111b6560428e93784
Time: 76066ms
chunk {app-pages-pages-module} app-pages-pages-module.js, app-pages-pages-module.js.map (app-pages-pages-module) 3.16 MB  [rendered]
chunk {main} main.js, main.js.map (main) 1.92 MB [initial] [rendered]
chunk {polyfills} polyfills.js, polyfills.js.map (polyfills) 492 kB [initial] [rendered]
chunk {runtime} runtime.js, runtime.js.map (runtime) 8.84 kB [entry] [rendered]
chunk {scripts} scripts.js, scripts.js.map (scripts) 1.32 MB  [rendered]
chunk {styles} styles.js, styles.js.map (styles) 3.99 MB [initial] [rendered]
chunk {vendor} vendor.js, vendor.js.map (vendor) 7.17 MB [initial] [rendered]
" %}   
{% include success.html content="You just compiled the UI based on Angular 7.0" %}

## Compose the microservices

At this point, we have all the necessary components. Let's put everything together by starting the different docker compositions. The order in which we start the compositions is 
important as there are dependencies:
- ``docker-compose-microservices.yml`` starts the [Kafka](https://kafka.apache.org/) message brocker and the microservices. This is what we already tested in [part 1](https://www.hostettler.net/2019/02/17/microservice-architecture-part-1.html) to prove that all the microservices are available.
- ``docker-compose-log.yml`` starts an [ElasticSearch, LogStash, and Kibana (ELK) suite](https://www.elastic.co/elk-stack) alongside a Logspout compagnon container to take care of logs. This aggregates **ALL** logs from all containers
and concentrate them into the ElasticSearch using Logstash. Kibana can then be used to analyze the logs and extract some intelligence, raise alerts and so on.
- ``docker-compose-api-gw.yml`` starts an api-gateway that routes the calls to the services and handle 
security by delegating authentication to a identity manager called [keyloak](https://www.keycloak.org/). It also serves static content and as [TLS termination](https://en.wikipedia.org/wiki/TLS_termination_proxy).


{% highlight bash %}
$ cd  docker-compose
$ docker-compose -f docker-compose-microservices.yml up &
$ docker-compose -f docker-compose-log.yml up &
$ docker-compose -f docker-compose-api-gw.yml up &
{% endhighlight %}
{% include console.html content="
counterparty-service    | 2019-03-12 20:06:27,203 INFO  [stdout] (default task-1)         counterpar0_.registrationStatus as registr16_0_,
counterparty-service    | 2019-03-12 20:06:27,203 INFO  [stdout] (default task-1)         counterpar0_.status as status17_0_
counterparty-service    | 2019-03-12 20:06:27,203 INFO  [stdout] (default task-1)     from
counterparty-service    | 2019-03-12 20:06:27,205 INFO  [stdout] (default task-1)         Counterparty counterpar0_
kafka                   | [2019-03-12 20:10:15,744] INFO [GroupMetadataManager brokerId=1] Removed 0 expired offsets in 0 milliseconds. (kafka.coordinator.group.GroupMetadataManager)
kafka                   | [2019-03-12 20:20:15,651] INFO [GroupMetadataManager brokerId=1] Removed 0 expired offsets in 0 milliseconds. (kafka.coordinator.group.GroupMetadataManager)
kafka                   | [2019-03-12 20:30:15,652] INFO [GroupMetadataManager brokerId=1] Removed 0 expired offsets in 0 milliseconds. (kafka.coordinator.group.GroupMetadataManager)
kafka                   | [2019-03-12 20:40:15,652] INFO [GroupMetadataManager brokerId=1] Removed 0 expired offsets in 0 milliseconds. (kafka.coordinator.group.GroupMetadataManager)
kafka                   | [2019-03-12 20:50:15,654] INFO [GroupMetadataManager brokerId=1] Removed 0 expired offsets in 0 milliseconds. (kafka.coordinator.group.GroupMetadataManager)
...
kibana           | {\"type\":\"response\",\"@timestamp\":\"2019-03-12T20:53:56Z\",\"tags\":[],\"pid\":1,\"method\":\"get\",\"statusCode\":302,\"req\":{\"url\":\"/\",\"method\":\"get\",\"headers\":{\"user-agent\":\"curl/7.29.0\",\"host\":\"localhost:5601\",\"accept\":\"*/*\"},\"remoteAddress\":\"127.0.0.1\",\"userAgent\":\"127.0.0.1\"},\"res\":{\"statusCode\":302,\"responseTime\":3,\"contentLength\":9},\"message\":\"GET / 302 3ms - 9.0B\"}
kibana           | {\"type\":\"response\",\"@timestamp\":\"2019-03-12T20:54:01Z\",\"tags\":[],\"pid\":1,\"method\":\"get\",\"statusCode\":302,\"req\":{\"url\":\"/\",\"method\":\"get\",\"headers\":{\"user-agent\":\"curl/7.29.0\",\"host\":\"localhost:5601\",\"accept\":\"*/*\"},\"remoteAddress\":\"127.0.0.1\",\"userAgent\":\"127.0.0.1\"},\"res\":{\"statusCode\":302,\"responseTime\":8,\"contentLength\":9},\"message\":\"GET / 302 8ms - 9.0B\"}
....
api-gateway         | 192.168.128.15 - - [12/Mar/2019:20:02:36 +0000] \"POST /plugins HTTP/1.1\" 409 213 \"-\" \"curl/7.29.0\"                                                                        
api-gateway         | 2019/03/12 20:02:36 [notice] 41#0: *139 [lua] init.lua:393: insert(): ERROR: duplicate key value violates unique constraint \"plugins_cache_key_key\"                       \" 
api-gateway         | Key (cache_key)=(plugins:oidc::::) already exists., client: 192.168.128.15, server: kong_admin, request: \"POST /plugins HTTP/1.1\", host: \"api-gateway:8001\"               
" %}   

If everything goes according to plan, you now have a working application ecosystem at ``https://localhost`` 
Point your browser to ``https://localhost`` and you'll get an nice UI. 
{% include image.html url="/figures/ui.png" description="Angular 7.0 UI to the financial-app" %}

Point it to the counterparty microservice at ``https://localhost/api/v1/counterparty``, the API-gateway will detect that you are not authenticated and will redirect you
to the SSO platform to enter for credentials. Enter ``user1/user1``
{% include image.html url="/figures/login.png" description="Keycloack SSO login form" %}
Once authenticated you get redirected to the orginal URL you requested (``https://localhost/api/v1/counterparty``)
{% include image.html url="/figures/counterparty-service.png" description="JSON result of the counterparty  microservice that returns all counterparties." %}

{% include success.html content="Kudos, you just completed the installation of a complete microservice ecosystem locally on your machine." %}

## Dissecting the docker-composes
As stated previously ``docker-compose`` composes several containers together to deliver a solution. 
For instance, by starting the database first and then whatever service that requires a database.
Using [docker-compose](https://docs.docker.com/compose/) you set the same parameters, environment variables, 
volumes that you would when starting a container with the command line.

From a general point of view, a docker-compose yaml file defines a series of services (e.g., database, microservice, web server) and then a series of "shared" services such as volumes, networks and so on.

Let's take the example of the `` docker-compose-api-gw.yml``  file. 
- First ``version "2.1"`` defines the version of the syntax. Then ``services`` defines a section with a series of services.
- In the below example, the first service is called ``kong-database`` and is based on a postgres database version 10 as stated by ``image: postgres:10``. The name of the container (for instance what
will appear if you run ``docker ps``) is ``kong-database``. The hostname will also be called ``kong-database``.
- After that, comes a section that describes the networks the container is participating into. This is very useful to isolate the containers from one another from a network perspective.
- The ``environment`` section defines environment variables (similar to ``-e`` in the command line). 
- The healthcheck section defines rules to state whether or not a container is ready for prime time
and heathly. 
- The ``kong-database`` example does not expose ports but it could so by defining a ``ports`` section that list the mapping of the ports of the container to the port of the host system. ``80:7070``  means
the the port ``7070`` of the container is mapped to the port ``80`` (http) of the host system.
- Finally, the volumes section maps volumes from the host systems to directory in the container. This is very useful to save the state of the container (e.g., database files)
or to put custom configurations in place.


{% highlight yaml %}
version: "2.1"

services:

   kong-database:
    image: postgres:10
    container_name: kong-database
    hostname: kong-database
    networks:
     - backend-network
    environment:
      POSTGRES_USER: kong
      POSTGRES_PASSWORD: kong
      POSTGRES_DB: kongdb
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "kong", "-d", "kongdb"]
      interval: 30s
      timeout: 30s
      retries: 3
    volumes:
      - pgdata-kong:/var/lib/postgresql/data
...
{% endhighlight %}

<br/>

With that very quick introduction to ``docker-compose`` let's have a look at the services delivered by the three ``docker-compose`` files of the demo:

#### ``docker-compose-log.yml``: Providing a logging infrastructure
Microservice architecture are distributed by nature and therefore cross-cutting concerns such as logging must take that aspect into account and aggregate the logs of the different containers.
Without that it would be difficult to follow a user request that goes accross many services to deliver the final value. 

To implement it, we rely on the [logspout](https://github.com/gliderlabs/logspout) log router. Logspout primarly captures all logs of all the running containers and route them to
a log concentrator. Logspout in itself does not do anything with the logs, it just routes them to something. In our case, that something is [Logstash](https://www.elastic.co/products/logstash).

Logstash is part of the ELK stack and is a pipeline that concentrate, aggregate, filters and stashes them in a database, usually [elasticsearch](https://www.elastic.co/products/elasticsearch).
Kibana depends on Elasticsearch and gets its configuration from a volumes shared from the host (``./elk-pipeline/``).
For more details about the Logstash configuration, please refer to ``./elk-pipeline/logstash.conf``.

[Elastic Search](https://www.elastic.co/products/elasticsearch) stores, indexes and searches large amount of data. Like Logstash, it is distributed in nature.
Elasticsearch is starting first in ``docker-compose-log.yml`` because other services such as Logstash and Kibana depends on it.
Elasticsearch maps a host volume (``esdata1``) to its own data directory (``/usr/share/elasticsearch/data``). Thanks to that mapping, data are not lost when the container is stopped or if it crashes.

The final part of the puzzle is [Kibana](https://www.elastic.co/products/kibana) which visualizes the data stored in Elasticsearch to do business intelligence on the logs. This is very 
useful to get a clear and real time status of the solution. Kibana depends on Elasticsearch and exposes its interface to the port ``5601`` of the host.

<br/>

#### ``docker-compose-api-gw.yml`` : Prodiving api-gateway services 
Microservice architecture are usually composed of a lot of services. Keeping track of these, providing and maintaining a clear API becomes very quickly challenging.
Besides, the granularity of microservices often call for a composition to deliver actual added value. Besides, different might have different needs. For instance, a mobile app might need a different API
that a web app.
Furthermore, we often want to secure some services. For instance, using [oauth2 protocol](https://oauth.net/2/) connected to an identity provider to offer Single Sign On (SSO) on the services.

In our case, the API gateway is called [Kong](https://konghq.com/solutions/gateway/) and it requires a database. The ``docker-compose-api-gw.yml`` describes the following services:

``kong-database`` which is a postgress database version 10 that holds the API gateway configuration

The api gateway itself ``api-gateway`` that is based on a docker image that we built previously ``unige/api-gateway``  when we ran ``mvn clean install -Ppackage-docker-image`` at the root of the project.
To get more details on how the image has been built, look at the ``Dockerfile`` in the ``api-gateway/src/main/docker`` directory. The image is based on ``kong:1.1rc1-centos`` but it is customized in several ways:
- There is an additional [plugin](https://github.com/nokia/kong-oidc) to support openid 
- A customized ``docker-entrypoint.sh`` to start Kong as root so that we can attach it to ports ``80`` and ``443`` that are privileged.
- A customer ``nginx`` template to enable serving static content (the UI).
- A shell script called ``config-kong.sh`` that configures the API-gateway by defining the services and the routes to these services. By the way this file, is ran after the api-gateway is
started and labelled as healthy by the container called ``api-gateway-init``.
The first line defines a service called ``counterparty-service`` that will route the request to the microservice ``http://counterparty-service:8080/counterparties``. The servicer ``counterparty-service`` is the host name
given by the microservice configuration. The second line creates a route in the API-gateway to the previous service. In that case ``/api/v1/counterparty``, please note that the api-gateway can
take care of versioning. Finally, the last line configures the OpenId plugin to provide authentication by telling the plugin to use the ``api-gateway`` client of the ``apigw`` realm of the keycloak.

{% highlight bash %}
#Creates the services.
curl -S -s -i -X POST --url http://api-gateway:8001/services --data "name=counterparty-service" --data "url=http://counterparty-service:8080/counterparties"
...
#Creates the routes
curl -S -s -i -X POST  --url http://api-gateway:8001/services/counterparty-service/routes --data "paths[]=/api/v1/counterparty" 
...
#Enable the Open ID Plugin
curl -S -s -i -X POST  --url http://api-gateway:8001/plugins --data "name=oidc" --data "config.client_id=api-gateway" --data "config.client_secret=798751a9-d274-4335-abf6-80611cd19ba1" --data "config.discovery=https%3A%2F%2Flocalhost%2Fauth%2Frealms%2Fapigw%2F.well-known%2Fopenid-configuration"
{% endhighlight %}

A database for Keycloak the SSO software called ``iam-db``

The SSO service ``iam`` that is based on Keycloak v4.8.3. Please note that a complete configuration is loaded initially using ``master.realm.json``. This configuration creates the required
realm, client and configuration to provide authentication to the api-gateway.

<br/>

#### ``docker-compose-microservices.yml`` : Micro-services and message brocker 
Finally, the last of the composition are the microservices themselves. As you can see, most of the configuration is not required for the microservices themselves but for the infrastructure around it.
All the services belong to the ``backend-network`` nertwork.

First, it defines a [ZooKeeper](https://zookeeper.apache.org/) that provides distributed configuration management, naming and group services. Zookeeper maintains its state in two shared volumes that are respectively mapped
to the  ``./target/zk-single-kafka-single/zoo1/data`` and ``./target/zk-single-kafka-single/zoo1/datalog`` directories of the host. Zookeeper is a mandatory component for the message broker.

Second, it defines a Kafka container. [Kafka](https://kafka.apache.org/) is a robust and fast message broker that excels at exchanging messages in a distributed way. It has a dependency to Zookeeper
and exposes its port ``9092`` to the same port on the host. It also saves its state on a mapped volume on the host.

Then, the counterparty service is a actual microservice (Finally !!!) that exposes its port ``8080`` to the port ``10080`` of the host. This microservice is based on [Thorntail](https://thorntail.io/).

The instrument service is special as it connects to the message broker (Kafka) to send messages that will be read later on by the valuation service.

The other microservices : valuation-service and regulatory-service are more of the same.

{% include success.html content="Kudos, you completed the tour of the microservice sample. Next chapter dives into a bit of theory." %}
