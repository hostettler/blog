---
layout: post
title: "Embedded Web Application integration testing using Tomcat 7 and JUnit"
date: 2012-04-09 14:39
comments: true
published: true
categories: [Integration testing, test, Web application, Java EE6, JEE6, Tomcat 7, WAR, JUnit]
keywords: [Integration testing, test, Web application, Java EE6, JEE6, Tomcat 7, WAR, JUnit]
description: "In this article, we look at how to start an embedded version of Tomcat in JUnit fixtures."
---

To follow up on my previous post about [how to programmatically build a web archive](/blog/2012/04/05/programmatically-build-web-archives-using-shrinkwrap/), I propose to look at how to deploy this archive in a Tomcat instance that is embedded into a JUnit test.

As usual, the presented snippets are available in the [JEE-6-Demo](http://code.google.com/p/jee6-demo/) application. I will first look at the dependencies, then I will show how to configure Tomcat and finally how to start and stop the embedded instance.

The first dependency is the embedded Tomcat core component. JULI which stands for Java Utility Logging Implementation that is the container extension of common logging. The ECJ compiler and JASPER are required to handle JSPs. Of course all these dependencies are scoped to test only. 

## The dependencies

```xml
<dependency>
	<groupId>org.apache.tomcat.embed</groupId>
	<artifactId>tomcat-embed-core</artifactId>
	<version>7.0.26</version>
	<scope>test</scope>
</dependency>
<dependency>
	<groupId>org.apache.tomcat.embed</groupId>
	<artifactId>tomcat-embed-logging-juli</artifactId>
	<version>7.0.26</version>
	<scope>test</scope>
</dependency>
<dependency>
	<groupId>org.eclipse.jdt.core.compiler</groupId>
	<artifactId>ecj</artifactId>
	<version>3.7.1</version>
	<scope>test</scope>
</dependency>
<dependency>
	<groupId>org.apache.tomcat.embed</groupId>
	<artifactId>tomcat-embed-jasper</artifactId>
	<version>7.0.26</version>
	<scope>test</scope>
</dependency>
<dependency>
    <groupId>junit</groupId>
    <artifactId>junit</artifactId>
    <version>4.10</version>
    <scope>test</scope>
</dependency>
``` 

## Configuring Tomcat
The following snippet prepares the embedded instance. The static member ``mTomcat`` references a new instance of Tomcat. As Tomcat still requires a file system to work, a temporary directory is used to put all the temporary files. To that end, we use the ``java.io.tmpdir`` java property.
These variables are shared among the test cases.

```java

/** The tomcat instance. */
private Tomcat mTomcat;
/** The temporary directory in which Tomcat and the app are deployed. */
private String mWorkingDir = System.getProperty("java.io.tmpdir");
```

As we want to use the same Tomcat configuration among all test cases, the initialization is put in a method annotated with ``@Before``. First, the method sets the port to ``0``. This tells the engine to choose the port to run on by itself.  This is especially useful to avoid  starting the embedded Tomcat on a already used port as ``8080`` for instance.
Then the base directory is set ``mTomcat.setBaseDir()`` to the temporary directory. Without doing that, Tomcat would start in the current directory. The rest of the method configures the way WAR are managed by the engine.

```java
@Before
public void setup() throws Throwable {
	mTomcat = new Tomcat();
	mTomcat.setPort(0);
	mTomcat.setBaseDir(mWorkingDir);
	mTomcat.getHost().setAppBase(mWorkingDir);
	mTomcat.getHost().setAutoDeploy(true);
	mTomcat.getHost().setDeployOnStartup(true);
	...
}
```

The rest of the method builds a reference to a directory (based into the temporary directory) that contains the web application. This directory is deleted if it exists to ensure redeployment of the WAR. Finally, the WAR built as explained [there](/blog/2012/04/05/programmatically-build-web-archives-using-shrinkwrap/) is exported into the temporary directory.

Finally, the web application is added to the Tomcat instance. More specifically, a path
exploded version ``webApp.getAbsolutePath()`` of the WAR is linked to a context ``contextPath``.

```java
@Before
public void setup() throws Throwable {
	...
	String contextPath = "/" + getApplicationId();
	File webApp = new File(mWorkingDir, getApplicationId());
	File oldWebApp = new File(webApp.getAbsolutePath());
	FileUtils.deleteDirectory(oldWebApp);
	new ZipExporterImpl(createWebArchive()).exportTo(new File(mWorkingDir + "/" + getApplicationId() + ".war"),
			true);
	mTomcat.addWebapp(mTomcat.getHost(), contextPath, webApp.getAbsolutePath());	
}
```

## Starting Tomcat
Now that Tomcat has been configured, the next step is to start it. We want a fresh Tomcat
for each and every test case. This way, a failed test does not have repercussions on the subsequent tests (e.g. session information, memory leaks);

```java
@Before
public void setup() throws Throwable {
    ...
	mTomcat.start();
}
```

If necessary, the actual port on which Tomcat has been started can be retrieved using the following snippet.
```java
protected int getTomcatPort() {
	return mTomcat.getConnector().getLocalPort();
}
```

Finally, after the test, the Tomcat instance is stopped and destroyed clearing the way for the next test.

## Stopping Tomcat
```java
@After
public final void teardown() throws Throwable {
	if (mTomcat.getServer() != null
            && mTomcat.getServer().getState() != LifecycleState.DESTROYED) {
        if (mTomcat.getServer().getState() != LifecycleState.STOPPED) {
        		mTomcat.stop();
        }
        mTomcat.destroy();
    }
}
```	

## Conclusion
In this post, we have seen how to start an embedded version of Tomcat into a JUnit fixture. 
The examples used in this blog are to be found in the [JEE-6-Demo](http://code.google.com/p/jee6-demo/) project on [Google code hosting](http://code.google.com).