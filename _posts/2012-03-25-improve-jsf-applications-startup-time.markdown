---
layout: post
title: "On JEE 6 webapps startup time"
date: 2012-03-25 23:20
comments: true
categories: [JSF, Java EE6, JEE6, performances, Tomcat 7, Weld, JUnit]
keywords: [JSF, Java EE6, JEE6, performances, Tomcat 7, Weld, JUnit]
description: "In this article, I present how to reduce the startup time of JEE6 web applications by doing minor changes in the configuration files."
---
While working on Tomcat 7 embedded to automate my integration tests, I realized that my integration tests wasted much of the time in starting/stopping the server. Even if I do not start the integration tests as often as the unit tests, it becomes rapidly irritating. Furthermore, during development I tend to restart the server a couple of times per hour, especially at the beginning of the project. Sure hot deployment helps, but it is not always enough.

On my Mac Book Pro, a cold start took around 10s. Interestingly enough,  an empty Tomcat startups in less than a second. The problem comes from the fact that Tomcat 7 scans the classpath to find out annotations that declare Servlets using the ``@WebServlet``. This, even if you do not use that feature. Don't get me wrong, not having to configure XML is cool but I am not ready to pay such a high price for it. Especially as the only servlet I use, is the JSF one.

Where do we start from?
-----------------------
For these tests, I use a JEE6 application with JSF, Weld and JPA (no EJBs) that runs under Tomcat.   
This is a demo application called [JEE-6-Demo](http://code.google.com/p/jee6-demo/) that I use to teach JEE6.
A mentioned previously, a cold start (without tuning anything) requires around 10s on my Mac Book Pro Intel Core i7 with 8Gb RAM : ``INFO: Server startup in 9992 ms``

Step 1: Avoid looking for ``@WebSerlet`` and co.
-----------------------------
By default, Tomcat 7 (along with the Servlet 3.0 specification) scans the classpath to look for classes that are annotated ``@WebServlet``,``@WebServletContextListener``, ``@ServletFilter``, or ``@InitParamJSF``.  It is a nice feature as you do not have to specify the faces servlet anymore.
However, it comes at a price: depending of the classpath this can be very long.
To solve this issues, simple add the ``metadata-complete="true"`` to the ``web-app`` element of our ``WEB-INF/web-xml`` attribute to avoid scanning the classpath.

```xml  
<web-app metadata-complete="true"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://java.sun.com/xml/ns/javaee"
	xmlns:web="http://java.sun.com/xml/ns/javaee/web-app_2_5.xsd"
	xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd"
	id="MyWebApp" version="3.0">
```

Obviously, as it is no more automatically discovered, we have  to manually add the faces servlet to the context:

```xml 
	<servlet>
		<servlet-name>Faces Servlet</servlet-name>
		<servlet-class>javax.faces.webapp.FacesServlet</servlet-class>
		<load-on-startup>1</load-on-startup>
	</servlet>

	<servlet-mapping>
		<servlet-name>Faces Servlet</servlet-name>
		<url-pattern>/faces/*</url-pattern>
	</servlet-mapping>
```

Using these modifications in ``web.xml``, the startup time came down to around 4.5 seconds:
``INFO: Server startup in 4404 ms``


Step 2: Avoid looking for ``@ManagedBean`` and co.
---------------------------------------------------
Similarly, the is a similar feature in JSF 2.0. By default, the JSF implementation looks for classes annotated with
As I use Weld and its ``@Named``, ``@SessionScoped``, and so on, I can disable this feature in JSF.
```xml  
<faces-config xmlns="http://java.sun.com/xml/ns/javaee"
   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
   xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-facesconfig_2_0.xsd"
   version="2.0"  metadata-complete="true">
```   

Using this modification in ``faces-config.xml``, the startup time came down to around 3.7 seconds:
``INFO: Server startup in 3730 ms``



Step 3: Limiting Weld's scanning
--------------------------------
Finally, I would like to keep Weld scanning to discover the ``@Named``, ``@Inject``, and other Weld annotations but I would like to limit it to my a subset of the classes of the jar. To that end simply add ``weld:scan`` directive and include a pattern with packages to scan.

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://java.sun.com/xml/ns/javaee" 
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
        xmlns:weld="http://jboss.org/schema/weld/beans" 
        xsi:schemaLocation="
           http://java.sun.com/xml/ns/javaee http://jboss.org/schema/cdi/beans_1_0.xsd
           http://jboss.org/schema/weld/beans http://jboss.org/schema/weld/beans_1_1.xsd">
  
<weld:scan>
    <weld:include pattern="ch.demo.*"/>
</weld:scan>
```

Using this modification in ``beans.xml``, the startup time came down to around 3.3 seconds:
``INFO: Server startup in 3312 ms``


To conclude, using these minor modifications I divided the startup time by three. This is very useful during development and integration tests when the server is started and stopped many times.