---
layout: post
title: "Integration Tests with Docker And Arquillian"
date: 2016-01-30 23:38:50 +0100
comments: true
categories:  JEE7, Tests, Integration Tests, Arquillian, Docker
---

Last month I decided to add a touch of [microservices](http://martinfowler.com/articles/microservices.html) to the JEE course I teach at the University of Geneva.
I ended up with a couple of microservices and as expected I came across the challenge of their integration testing.
This post is **NOT** specifically about microservices. It rathers focuses on the JEE integration testing experience I encountered while building the microservices. A dedicated blog post will follow on my journey through the building of microservices with JEE. 

## A short description of the architecture

From a technological perspective, I am using JEE 7 on [Wildfly](http://wildfly.org/) with [MySql](http://wildfly.org/). Therefore, my microservices are wars composed of 1-2 EJBs plus a restful service that exposes the logic. Typically, my microservices are composed of 4-5 classes of max 150 lines of codes each. On my laptop, a microservice deploys in less than 5 seconds. I mainly need to test EJBs and their database calls. I also want to test integration between microservices.

### Why Docker?
As it serves as a example for a course, I want it to be super easy to install/re-install 20 times if necessary. Furthermore, I want fast-paced deployment. To that end, Docker is a great tool because I do not have to bother on what laptop/computer the students work (provided they can run [Docker Toolbox](https://www.docker.com/products/docker-toolbox)). Moreover, all the tools/middlewares I am using for the course are already packaged as Docker images.


## Building a  Docker image for a Wildfly Integration Test Server
As mentioned previously, I propose to use Wildfly +  MySql as the runtime environment. However, at test time, I do not want to start both an application server and a database. More important, I want to get a fresh database for each and every test suite. Futhermore, I would like to use on a simpler setup for the application server. For instance, I do prefer to use an in memory H2 database instead of MySql. I also do not want to bother with LDAP/JAAS configuration, clustering, etc....
Of course, datasource name, realm name and more generally all the resources required by the microservices must be present with their production name.

Ones of Wildfly's great features is its ability to be configured using the command line.
The first step is to configure a data source relying on H2 that has the same name as the production one.

```bash
# First step : Add the datasource
data-source add --name=StudentsDS --driver-name=h2 --jndi-name=$STUDENTS_DS --connection-url=$H2_URI --user-name=$H2_USER --password=$H2_PWD --use-ccm=false --max-pool-size=25 --blocking-timeout-wait-millis=5000
```
Then, let's configure a realm.  This is helpful when integration tests rely on principals and do verify the security.
The file ``jee7-demo-realm-users.properties`` (resp. ``jee7-demo-realm-roles.properties``) defines the users (resp. the roles) of the realm.

```bash
# Add a property file realm
/subsystem=security/security-domain=jee7-demo-realm:add(cache-type=default)
/subsystem=security/security-domain=jee7-demo-realm/authentication=classic:add()
/subsystem=security/security-domain=jee7-demo-realm/authentication=classic/login-module=UsersRoles       \
    :add(code=UsersRoles, flag=required,                                                        \
         module-options={"usersProperties"=>"${JBOSS_CUSTOMIZATION}/jee7-demo-realm-users.properties",   \
                         "rolesProperties"=>"${JBOSS_CUSTOMIZATION}/jee7-demo-realm-roles.properties"})
```
Finally, let's add an admin user in order to allows Arquilian or an IDE to interact with this application server.
```bash
/opt/jboss/wildfly/bin/add-user.sh admin admin
```


Hereafter the complete configuration ``config_wildfly.sh``.

```bash
#!/bin/bash

# Usage: execute.sh [WildFly mode] [configuration file]
#
# The default mode is 'standalone' and default configuration is based on the
# mode. It can be 'standalone.xml' or 'domain.xml'.

JBOSS_HOME=/opt/jboss/wildfly
JBOSS_CUSTOMIZATION=$JBOSS_HOME/customization
JBOSS_STANDALONE_CONFIG=$JBOSS_HOME/standalone/configuration/
JBOSS_CLI=$JBOSS_HOME/bin/jboss-cli.sh
JBOSS_MODE=${1:-"standalone"}
JBOSS_CONFIG=${2:-"$JBOSS_MODE.xml"}

function wait_for_server() {
  until `$JBOSS_CLI -c ":read-attribute(name=server-state)" 2> /dev/null | grep -q running`; do
    sleep 1
  done
}

echo "=> Starting WildFly server"
$JBOSS_HOME/bin/$JBOSS_MODE.sh -b 0.0.0.0 -c $JBOSS_CONFIG &

echo "=> Waiting for the server to boot"
wait_for_server

echo "=> Executing the commands"
export STUDENTS_DS="java:/StudentsDS"
export H2_URI="jdbc:h2:mem:STUDENTS_DB;DB_CLOSE_DELAY=-1"
export H2_USER="sa"
export H2_PWD="sa"

$JBOSS_CLI -c << EOF
batch

echo "Connection URL: " $CONNECTION_URL

# First step : Add the datasource
data-source add --name=StudentsDS --driver-name=h2 --jndi-name=$STUDENTS_DS --connection-url=$H2_URI --user-name=$H2_USER --password=$H2_PWD --use-ccm=false --max-pool-size=25 --blocking-timeout-wait-millis=5000 

# Then configure a realm that relies on property files
/subsystem=security/security-domain=jee7-demo-realm:add(cache-type=default)
/subsystem=security/security-domain=jee7-demo-realm/authentication=classic:add()
/subsystem=security/security-domain=jee7-demo-realm/authentication=classic/login-module=UsersRoles       \
    :add(code=UsersRoles, flag=required,                                                        \
         module-options={"usersProperties"=>"${JBOSS_CUSTOMIZATION}/jee7-demo-realm-users.properties",   \
                         "rolesProperties"=>"${JBOSS_CUSTOMIZATION}/jee7-demo-realm-roles.properties"})

# Execute the batch
run-batch
EOF

# Finally, let's add an admin that can be used by the IDE to deploy the tests
/opt/jboss/wildfly/bin/add-user.sh admin admin 

echo "=> Shutting down WildFly"
if [ "$JBOSS_MODE" = "standalone" ]; then
  $JBOSS_CLI -c ":shutdown"
else
  $JBOSS_CLI -c "/host=*:shutdown"
fi
```

The next step is to enhance the official Wildfly image called ``jboss/wildfly:latest`` with the specific configurations required for integration testing. This following Dockerfile describes how to build the image that we will use for testing.
First, it adds the previous configuration files ``./config_wildfly.sh ./jee7-demo-realm-roles.properties ./jee7-demo-realm-users.properties`` to the ``/opt/jboss/wildfly/customization/`` of the image. Then we tell Docker to run the configuration ``config_wildfly.sh`` and to do some cleanup. After what, it will record the states as a new image.


```
FROM jboss/wildfly:latest

ADD ./config_wildfly.sh ./jee7-demo-realm-roles.properties ./jee7-demo-realm-users.properties /opt/jboss/wildfly/customization/

RUN ["/opt/jboss/wildfly/customization/config_wildfly.sh"]
RUN rm -rf  /opt/jboss/wildfly/standalone/configuration/standalone_xml_history
CMD ["/opt/jboss/wildfly/bin/standalone.sh", "--debug", "-b", "0.0.0.0", "-bmanagement", "0.0.0.0"]
EXPOSE 8787
```
Now we can build an image called ``jee7-test-wildfly`` using the following command (in the directory where the ``Dockerfile`` lives):
```bash
docker build --no-cache -rm -t jee7-test-wildfly .
```


The following command runs the image we just built, exposes (and map) the port 8080, 9090 and 8787, and mount the local directory ``/Users/XXXXXXXXXX/tmp/docker-deploy`` on the image's ``/opt/jboss/wildfly/standalone/deployments/`` directory. This is of course the directory in which the integration tests are to be deployed..

```bash
docker run -d  -p 8080:8080 -p 9990:9990 -p 8787:8787 -v /Users/XXXXXXXXXX/tmp/docker-deploy:/opt/jboss/wildfly/standalone/deployments/:rw jee7-test-wildfly
```

Now that we do have a container running an application server with in memory database and a simple realm, we can configure the test harness.

## How to configure Arquillian
[Arquillian](http://arquillian.org/) is a JEE integration test framework. It allows to test JEE components such as EJBs or web services.
The first step it to tell Arquillian where the Wildfly server lives. The following ``arquillian.xml`` file states that the integration tests should deploy on a Wildfly container that listens at ``192.168.99.100`` (which is the Docker container address) on port ``9990`` (which is the administration port). Furthermore, it declares the admin username and password. 


```xml
<?xml version="1.0"?>
<arquillian xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xmlns="http://jboss.org/schema/arquillian"
	xsi:schemaLocation="http://jboss.org/schema/arquillian
  http://jboss.org/schema/arquillian/arquillian_1_0.xsd">

	<container qualifier="wildfly" default="true">
		<configuration>
			<property name="managementAddress">192.168.99.100</property>
			<property name="managementPort">9990</property>
			<property name="username">admin</property>
			<property name="password">admin</property>
		</configuration>
		<protocol type="Servlet 3.0">
			<property name="host">192.168.99.100</property>
			<property name="port">8080</property>
		</protocol>
	</container>

	<extension qualifier="jacoco">
		<property name="includes">ch.demo.*</property>
	</extension>
</arquillian>
```

## A simple integration Test

Let us now write a simple integration test. First, we must tell Arquillian the it is in charge of running the test (``@RunWith(Arquillian.class)``). 

```java
@RunWith(Arquillian.class)
public class StudentServiceImplTest {
```
To test a given component (let's say an EJB), arquillian expects a well-formed Java component (either a jar or a war).
The following test is composend of a package contaning the EJB under test (``ch.demo``), an empty ``beans.xml`` to enable CDI and finally a ``persistence.xml`` to enable JPA.

```java
@Deployment
public static JavaArchive create() {
   return ShrinkWrap.create(JavaArchive.class, "integration-test-demo.jar").addPackages(true, "ch.demo")
      .addAsManifestResource(EmptyAsset.INSTANCE, "beans.xml")
      .addAsManifestResource("test-persistence.xml", "persistence.xml");
}
```
The previous JAR as well as the tests are packaged as a WAR and deployed on the application server declared in the ``arquillian.xml`` file. The following test injects the EJB under test implementing the ``StudentService`` interface and tests its ``add`` method.

```java
@Inject
StudentService service;

@Test
public void shouldAddReturnAllWithTheNewStudent() {
   Integer nbStudentsBeforeTest = service.getNbStudent();
   service.add(new Student("Doe", "Jane", new Date(), new PhoneNumber("+33698075273")));
   Integer nbStudentsAfterTest = service.getAll().size();
   Assert.assertSame(nbStudentsBeforeTest + 1, nbStudentsAfterTest);
   Assert.assertEquals(service.getAll().get(nbStudentsAfterTest - 1).getLastName(), "Doe");
}
```

## Maven, IDE Integration and Coverage
Finally, let me add that it is possible to get coverage data from Arquillian by enabling extensions. In the following, it enables```jacoco``. This produces coverage data that can be used by the Eclipse ECL-EMMA plugin.

```xml
<extension qualifier="jacoco">
   <property name="includes">ch.demo.*</property>
</extension>
```


## Conclusion
Docker and Arquillian provide a nice and seamless way for JEE integration testing. Nevertheless, I had a hard time at the beginning because Arquillian error handling in case of undeployable test archive is not very good. In this case, make sure that you package your test correctly (in the method annotated ``@Deployment``). In particular, double check ``beans.xml``, ``web.xml`` and the JAR/WAR structure. It really helped me to unzip the deployed test archive to figure out what when wrong in my code.
