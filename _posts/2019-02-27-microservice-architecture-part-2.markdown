---
layout: post
title: "Microservice Architecture - Part 2 (SSO, Logging, and all that)"
date: 2019-02-17T22:51:20+01:00
published: true
---

In [part 1](https://www.hostettler.net/2019/02/17/microservice-architecture-part-1.html), we discussed how to compile and deploy the microservices. Remember that the microservices themselves are only a part of the microservice architecture. 
By its very nature, microservice architecture are distributed and that comes with a lot of benefits and some constraints.
One of these constraint is that all the non-functional features such as security, logging, testability and so on, have to take distribution into account.

## Getting the backend components to run

If you wish to generate new certificates, you must regenerate the docker image ``unige/web-sso`` . To do so, run the appropriate maven command:
{% highlight bash %}
$ mvn clean install -Ppackage-docker-image
{% endhighlight %}
{% include console.html content="
...
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  07:22 min
[INFO] Finished at: 2019-03-06T21:18:13+01:00
[INFO] --------------------------------------------------------------------
" %} 


Now, let's compile the UI
{% highlight bash %}
$ cd web-ui
$ node --version
{% endhighlight %}
{% include console.html content="
6.5.0
" %}   
{% highlight bash %}
$ npm --version
{% endhighlight %}
{% include console.html content="
10.15.0
" %}   
{% highlight bash %}
$ npm install
{% endhighlight %}
{% include console.html content="
...
audited 31887 packages in 68.922s
" %}   

{% highlight bash %}
$ npm build
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

At this point, we have all the necessary components. Let's put everything together by starting the different docker compositions. The order in which we start the compositions is 
important as there are dependencies:
- ``docker-compose-microservices.yml`` starts the [Kafka](https://kafka.apache.org/) message brocker and the microservices. This is what we start in part 1 to test that all the microservices are available.
- ``docker-compose-log.yml`` starts an [ElasticSearch, LogStash, and Kibana (ELK) suite](https://www.elastic.co/elk-stack) alongside a Logspout compagnon container to take care of logs. This will ALL logs from all containers
and concentrate them into the ElasticSearch using Logstash. Kibana can then be used to analyze the logs and extract intelligence.
- ``docker-compose-api-gw.yml`` starts an api-gateway that will route the calls to the services and handle security by delegating authentication to a indendity manager called [keyloak](https://www.keycloak.org/). It will also serve static content and serve as [TLS termination](https://en.wikipedia.org/wiki/TLS_termination_proxy).


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

If everything goes according to plan, you know have a working application ecosystem at ``https://localhost`` 
Point your browser to ``https://localhost`` and you'll get an nice UI. 
{% include image.html url="/figures/ui.png" description="Angular 7.0 UI to the financial-app" %}

Point it to the counterparty microservice at ``https://localhost/api/v1/counterparty``, the API-gateway will detect that you are not authenticated and will redirect you
to the SSO platform to be asked for credentials. Enter ``user1/user1``
{% include image.html url="/figures/login.png" description="Keycloack SSO login form" %}
Once authenticated you get redirected to the orginal URL you requested (``https://localhost/api/v1/counterparty``)
{% include image.html url="/figures/counterparty-service.png" description="JSON result of the counterparty  microservice that returns all counterparties." %}

{% include success.html content="Kudos, you just completed the installation of a complete microservice ecosystem locally on your machine." %}