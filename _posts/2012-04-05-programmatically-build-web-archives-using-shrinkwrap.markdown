---
layout: post
title: "Programmatically build web archives using ShrinkWrap"
date: 2012-04-05 09:09
comments: true
categories: [ShrinkWrap, Java EE6, JEE6, Tomcat 7, WAR]
keywords: [ShrinkWrap, Java EE6, JEE6, Tomcat 7, WAR]
description: "In this blog post, I show how to programmatically build a web archive for test purposes."
---

While doing integration tests on web applications, it is important to restrict the application to the strict minimum necessary for any given test. It helps to improve the startup time as well as to reduce unexpected interactions.
To that end, I use [ShrinkWrap](http://www.jboss.org/shrinkwrap) from [JBoss](http://www.jboss.org/) that is a very nice a simple API to build Java archives (e.g., JAR WAR, EAR). This post describes how to programmatically build a WAR file that can be deployed on an embedded server for integration
testing. For more details on how to use this web archive in JUnit fixtures please refer to this [article](/blog/2012/04/09/embedded-jee-web-application-integration-testing-using-tomcat-7/). The following snippet gets the ShrinkWrap package from the JBoss repository. As I only use it for test purposes, I restricted its use to the test scope.

**ShrinkWrap** is available as a Maven dependency:
```xml
<repositories>
...
	<repository>
		<id>repository.jboss.org</id>
		<name>JBoss Repository</name>
		<url>http://repository.jboss.org/nexus/content/groups/public-jboss/</url>
	</repository>
	...
</repositories>
...
<dependencies>
	...
	<dependency>
		<groupId>org.jboss.shrinkwrap</groupId>
		<artifactId>shrinkwrap-api</artifactId>
		<version>1.0.0-alpha-12</version>
		<scope>test</scope>
	</dependency>
	<dependency>
		<groupId>org.jboss.shrinkwrap</groupId>
		<artifactId>shrinkwrap-impl</artifactId>
		<version>1.0.0-alpha-12</version>
		<scope>test</scope>
	</dependency>
...
<dependencies>
```

## Building the archive

The first step is to crate a web archive a.k.a. a WAR file. As we want to produce a WAR file, we use the ``WebArchive.class`` parameter
to the ``ShrinkWrap.create`` method. It is also possible to produce JAR (``JavaArchive.class``) and EAR (``EnterpriseArchive.class``).
```java  
WebArchive archive = ShrinkWrap.create(WebArchive.class, "test.war");
```
The next step to build a WAR is to provide the ``web.xml`` for you project. Again this can a be a simplified version of the web descriptor to improve startup time or limit dependencies. In the following example, the files are taken from a maven directory structure.

```java  
private static final String WEBAPP_SRC = "src/main/webapp";

archive.setWebXML(new File(WEBAPP_SRC, "WEB-INF/web.xml"))
```
Now that we have a web descriptor, let's add some classes that are required to run the application. To not having to add each single class individually, it is possible to a complete package to the archive.
```java  
archive.addPackage(java.lang.Package.getPackage("ch.demo.web"))
```

It is also possible to add individual classes:
```java 
archive.addClasses(Student.class, Address.class);
```

For JSF applications, we add add the ``faces-config.xml``. 
```java
archive.addAsWebInfResource(new File(WEBAPP_SRC, "WEB-INF/faces-config.xml"))
```

And similarly, let's add the CDI descriptor into ``WEB-INF``. As we do not need to create a specific ``beans.xml`,
we add an empty descriptor:
```java
archive.addAsWebInfResource(EmptyAsset.INSTANCE, "beans.xml")
```

Let's finally add standard web resources such as ``xhtml`` files or static files:
```java
archive.addAsWebResource(new File(WEBAPP_SRC, "login.xhtml"));
```

As ShrinkWrap uses [Method Chaining](http://en.wikipedia.org/wiki/Method_chaining), it is possible to chain ``add`` class to fill the archive.
```java  
archive.setWebXML(new File(WEBAPP_SRC, "WEB-INF/web.xml"))
	.addAsWebInfResource(EmptyAsset.INSTANCE, "beans.xml")
	.addAsWebInfResource(new File(WEBAPP_SRC, "WEB-INF/faces-config.xml"))
	.addPackage(java.lang.Package.getPackage("ch.demo.web"))
	.addAsWebResource(new File(WEBAPP_SRC, "login.xhtml"))
	.addAsWebResource(new File(WEBAPP_SRC, "index.jsp"))
	.addAsWebResource(new File(WEBAPP_SRC, "xhtml/listStudents.xhtml"), "xhtml/listStudents.xhtml");
```

Please note that when using maven to execute the tests, most of the required libraries are already on the classpath. Therefore, we often do not need to add any library or jars to the web archive.

## Using the archive

After having built the package, we need to export it either as a war or as an exploded directory.
The following snippet produces a WAR file named after the archive. The second parameter states whether
a existing archive can be overwritten.
```java 
new ZipExporterImpl(archive).exportTo(new File(archive.getName()), true);
```
Alternatively it is possible to produce an exploded archive:
```java  
new ExplodedExporterImpl(archive).exportExploded(new File(archive.getName()));
```

## Conclusion
We have seen how to use ShrinkWrap to programmatically build Web archives. This is especially useful to test smaller version of your web application during integration testing. It helps to test webapps with mock services and to improve the startup time. The examples used in this blog are to be found in the [JEE-6-Demo](http://code.google.com/p/jee6-demo/) project on [Google code hosting](http://code.google.com).
