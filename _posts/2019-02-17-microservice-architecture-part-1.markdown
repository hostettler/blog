---
layout: post
title: "Microservice Architecture - Part 1"
date: 2019-02-17T22:51:20+01:00
---


## Introduction
This series of blog posts aims at helping students at the University of Geneva to develop their first application following micro-service principles. 
Besides explaining the concepts and implementation details of micro-service architecture, we will as well discuss software development practices such as software 
factories and innovative deployment options such as containers. All samples and a complete working application can be found [here on GitHub](https://github.com/hostettler/microservices.git)

# Pre-requisites

{% include info.html content="This series of blog post leverages a lot of different technologies. Please take the time to install everything properly. It will save time later on." %}

To execute the samples you will need to install and to configure the following tools:
- a "reasonnably" powerfull computer with Linux (whatever recent distribution) or Windows (min. Windows 10) to support Docker. Mac is ok as well but it requires some additional steps that will not be described here.
- a working Docker environnment to deploy the services locally.
- a [JDK 11](https://jdk.java.net/11/) to compile and run the services
- a Git client for collaboration
- [NodeJS](https://nodejs.org/) and the [Angular tooling](https://angular.io/guide/quickstart)
- [Apache Maven](https://maven.apache.org/) for the automation.
- a bash interpreter (on Windows you can rely on Git bash that is usually installed with Git)

{% include warning.html content="because of a bug in one of the 3rd party software, please use Apache Maven 3.5.x instead of the latest 3.6.x. For more information pleaser refer to the Thorntail issue : https://github.com/thorntail/thorntail/pull/1195" %}

On top of that you need to have:
- A intermediate level in Java
- Some basic understanding of OS (including bash scripting) and networking (DNS, TCP, HTTP)
- a great deal of patience and coffee

## Getting everything to run
First thing first, let's checkout the code and compile everything
Let's clone the code from [GitHub](https://github.com/hostettler/microservices.git). 

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


The next step is to compile the project to get the artifacts (i.e., binaries). To that end, 
we use Apache Maven that is opiniated build tool. For more information and tutorials please refer to this [Maven Tutorial](https://maven.apache.org/guides/getting-started/maven-in-five-minutes.html)
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

At this point, you compiled all of the Java code and you created maven artifacts for each micro-service (Java Archives a.k.a. JARs). But as we will see in the next chapters, a micro-service architecture is much more than a bunch of micro-services. We will need a lot of additional 3rd party tools and services.
These additional services (e.g., logging, security) are usually provided as container images that runs on Docker. Therefore, we will run everything in Docker (more of Docker in this chapter)
Therefore, we will package our micro-services as docker images. Docker images are merely lightweight Linux systems with additional softwares. For more about [Docker](https://www.docker.com/)
Let's first check whether docker is properly installed.
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

So the docker daemon is up and running. Let's create the docker images for the micro-services.
{% highlight bash %}
mvn install -Pdocker
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

All the docker images for the micro-services have been created. Let's double check:
{% highlight bash %}
$ docker image ls | grep unige
{% endhighlight %}
{% include console.html content="
unige/regulatory-service                 latest              d19aed8e7179        About a minute ago   761MB
unige/valuation-service                  latest              38d95425c608        2 minutes ago        795MB
unige/instrument-service                 latest              cbd3898a097b        2 minutes ago        795MB
unige/counterparty-service               latest              e58b75e7fc43        3 minutes ago        762MB
" %}

{% include success.html content="You now have Docker images for your microservices" %}

We have a hostsname, no let's generate the SSL certificates so that we can encrypt the data in transit. This step is optional as you can reuse the dummy certificates in the source directory.



We want the application to be accessible by a the following name ```financial-app.com```. To that end we need the docker host ip address to be resolved to this hostname and vice-versa.
First, we must identify the docker host ip address. On windows, run the command ```ipconfig```. On Linux, run the command ```ifconfig```. Look for an adapter called DockerNAT (or something similar).
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


We have a hostsname, no let's generate the SSL certificates so that we can encrypt the data in transit. This step is optional as you can reuse the dummy certificates in the source directory.
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


No, let's compile the UI
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

# Bibliography
