---
layout: post
title: "Microservice Architecture - Part 2 (SSO, Logging, and all that)"
date: 2019-02-17T22:51:20+01:00
published: false
---

In part 1, we discussed how to compile and deploy the microservices. Remember that the microservices are only a part of the microservice architecture. By its very nature, microservice architecture are distributed and that comes with a lot of benefits and some constraints.
One of these constraint is that all the non-functional features such as security, logging, testability and so on, have to take distribution into account.

## Getting the backend components to run

We want the application to be accessible by a the following name ```financial-app.com```. To that end we need the docker host ip address to be resolved to this hostname 
and vice-versa. First, we must identify the docker host ip address. On windows, run the command ```ipconfig```. On Linux, run the command ```ifconfig```. 
Look for an adapter called DockerNAT (or something similar).

{% highlight bash %}
$ ipconfig
{% endhighlight %}
{% include console.html content="
Windows IP Configuration
....
Ethernet adapter vEthernet (DockerNAT) 2:

   Connection-specific DNS Suffix  . :
   Link-local IPv6 Address . . . . . : fe80::9d3f:55b9:4504:323f%19
   IPv4 Address. . . . . . . . . . . : 192.168.240.1
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Default Gateway . . . . . . . . . :
" %}   


So the ip-address is ```192.168.240.1```, let's add it to the hosts file. On Windows that file is located at ```C:\Windows\System32\drivers\etc``` and on Linux at ```/etc/hosts```

{% highlight bash %}
$ cat C:\Windows\System32\drivers\etc
{% endhighlight %}
{% include console.html content="
192.168.240.1 host.docker.internal
192.168.240.1 gateway.docker.internal
192.168.240.1 financial-app.com
" %}   

We have a hostsname, no let's generate the SSL certificates so that we can encrypt the data in transit. This step is optional as you can reuse the dummy certificates (for test purposes only)
in the source directory. This will create a certificate for ``financial-app.com``.
{% highlight bash %}
$ cd web-sso-docker
$ cd certs
$ openssl req -new -newkey rsa:4096 -days 365 -nodes -x509 -subj "/C=CH/ST=Geneva/L=Geneva/O=Dis/CN=financial-app.com" -keyout financial-app.com.key -out financial-app.com.crt
{% endhighlight %}
{% include console.html content="
Generating a RSA private key
.......................................................++++
......++++
writing new private key to 'financial-app.com.key'
-----
" %}   
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
- ``docker-compose-microservices.yml`` starts the Kafka message brocker and the microservices. This is what we start in part 1 to test that all the microservices are available.
- ``docker-compose-log.yml`` starts an ElasticSearch, LogStash, and Kibana (ELK) suite alongside a Logspout compagnon container to take care of logs. This will ALL logs from all containers
and concentrate them into the ElasticSearch using Logstash. Kibana can then be used to analyze the logs and extract intelligence.
- ``docker-compose-api-gw.yml`` starts an api-gateway that will route the calls to the services and handle security by delegating authentication to a indendity manager called keycloak.
- ``docker-compose-sso.yml`` 


{% highlight bash %}
$ docker-compose -f up
{% endhighlight %}
{% include console.html content="

" %}   


# Bibliography
