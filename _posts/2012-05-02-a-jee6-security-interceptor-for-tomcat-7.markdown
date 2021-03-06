---
layout: post
title: "A JEE6 security interceptor for Tomcat 7"
date: 2012-05-02 18:23
comments: true
categories: [Security, Java EE 6, JEE6, Java, JAAS, Interceptor, Tomcat 7]
keywords: [Security, Java EE 6, JEE6, Java, JAAS, Interceptor, Tomcat 7]
---

Security in Java EE is a well-known subject and JAAS, the framework that manages security, is
stable and reliable. Nevertheless, its API is quite low level and not always very convenient. For
that reason, there exist a number of frameworks that have been built on top of JAAS that provide
nice abstractions and convenient tools. This is especially true for the JEE application servers such as [Glassfish](http://glassfish.java.net/), [JBoss](http://www.jboss.org/) and the like.

In servlet containers such as [Tomcat](http://tomcat.apache.org/) or [Jetty](http://jetty.codehaus.org/jetty/) however, only the web components are secured and it is not trivial to secure services and the data access layer. This can be solved by integrating framework such as [Spring Security](http://static.springsource.org/spring-security/site/), [Apache Shiro](http://shiro.apache.org/).

As always it is not always possible to put a new framework or library in place. I will not
discuss whether it is a good or a bad idea to develop a new security mechanism. Nevertheless, a good
rule of thumb is to reuse existing components and not reinvent the wheel ... when possible. If for
let's say licensing reasons, it is not possible: here is a way to provide a simple declarative
authorisation procedure based on JAAS and the JEE6 interceptors. The objective is to propagate the
authorisation to non-web layers such as services or data access layer through a thread-local
variable.

The goal is to be able to write something similar to the following snippet. The idea is to annotate
a method (or a class) with a list of roles that are authorised. If the method executes in a context
in which a user has not enough rights, it raises an exception.

```java 
@Secure(roles = { "user" })
public List<Student> getAll() {
    ...
}

@Secure(roles = { "admin" })
public void add(final Student student) {
    ...
}
```

### Java Authentication and Authorisation System
Java Authentication and Authorisation System (JAAS) provides a security infrastructure for JAVA.
Both **authentication** (are you who you pretend to be?) and **authorisation** (are you allowed to do something you would like to do?) are covered. JAAS relies on the concept of principal. 
A principal is a particular identity of a user (e.g. social security id, driver licence id, ...).
By default, in JAVA SE, there is no easy way to have the list of roles granted to a given user.
To solve that problem, we need a bean that knows to which role a given Principal is authorised.
This is the role of ``MyPrincipal``.

```java 
public class MyPrincipal implements Principal, Serializable {

    public MyPrincipal(final Principal pPrincipal, final List<String> pRoles) {
        this.principal = pPrincipal;
        this.roles = pRoles;
    }
    ...  
    public boolean isUserInRole(final String pRole) {
        return roles.contains(pRole);
    }
}
```

### JEE6 Interceptors
Since JEE6, a feature called interceptors enables simple 
[Aspect Oriented Programming](http://en.wikipedia.org/wiki/Aspect-oriented_programming) without
using any specific framework. It works, of course, only on managed classes. Namely, classes that are instantiated by [CDI](http://docs.oracle.com/javaee/6/tutorial/doc/giwhb.html) through dependency injection. 

Just to fix the vocabulary, here some definition about Aspect Oriented Programming:

- Advice: code at it executed at a certain point in the code. This point is called a join point;
-  Join point: place in the code at which a given advice is executed.; 
- Pointcut: set of join point, usually described by a meta data that can be internal or external to the program (xml, annotations);
- Aspect: advice + its pointcut;

In Java EE6, an interceptor is made of an annotation that characterizes an aspect and an
implementation for that annotation. The annotation is placed at a join point, that is a place in the code where a specific advice must be executed. Usually around a method call. However, JEE6
interceptors are not as powerful as advanced AOP frameworks such as AspectJ, it provides enough
facility to decouple non-functional behaviors from the business code. That is exactly what we want
to achieve here. Let us now illustrate JEE6 interceptors through the implementation of a declarative security annotation that protects calls to the service layer.


#### A meta data that asks for security
The first thing is to declare an annotation that is parametrized by an array of ``String`` that
represents roles. This is presented in the following snippet. The annotation ``InterceptorBinding``
is required to mark the annotation as a pointcut. The `` @Retention(RetentionPolicy.RUNTIME)``
declares that Then ``@Target({ ElementType.METHOD, ElementType.TYPE })`` tells that the annotation
can be applied at both method level and type level. Finally, the member ``String[] roles();``
declares that the annotation has a parameter called ``roles``.
 
```java 
@InterceptorBinding
@Retention(RetentionPolicy.RUNTIME)
@Target({ ElementType.METHOD, ElementType.TYPE })
public @interface Secure {
    @Nonbinding
    String[] roles();
}
```

#### The interceptor (advice) that check the authorisation
The next step is to program an advice. To that end, a
simple class must be annotated by the annotation ``@Interceptor`` and by the annotation that it
implements ``@Secure(roles = { })`` (i.e. the pointcut). The method that is annotated by 
``@AroundInvoke`` is executed by the container when it encounters a method annotated by ``@Secure``
(i.e. join point). The name of the method (e.g. ``invoke`` ) does not matter as long as it returns an 
``Object`` and it accepts an ``InvocationContext`` as a parameter. This context contains information
about the intercepted method.
 
About the method itself, ``getRoles`` is a private method (given hereafter) that extract the list
of roles that are authorised to this method. The ``SecurityContext`` is described below and is
container for the ``ThreadLocal`` variable that contains the principal of the current user. The
method ``principal.isUserInRoles(roles)`` returns true if one the roles matches the expected
authorisation. If it is not the case an exception is raised and the interceptor stops there without
having executed the intercepted method. Otherwise, the interceptor goes on and executes the
intercepted method and finally returns its return value by executing ``return context.proceed()`` .

```java  
@Secure(roles = { })
@Interceptor
public class SecurityInterceptor implements Serializable {  

    @AroundInvoke
    public Object invoke(final InvocationContext context) throws Exception {
        String[] roles = getRoles(context.getMethod());
        MyPrincipal principal = SecurityContext.getPrincipal();

        if (!principal.isUserInRoles(roles)) {
            throw new IllegalAccessException("Current user not autorised!");
        }

        return context.proceed();
    }

    private String[] getRoles(final Method method) {
        ...
    }
}
```

The private method getRoles scans the class of the method that has been intercepted in order to
discover the list of roles. First it scans at method level (``method.isAnnotationPresent`` ) to
find the annotation `` Secure.class`` that has been described above. If it does not find the proper
annotation then it scans at class level (``method.getDeclaringClass().isAnnotationPresent`` ).
Remember that we allow to put the annotation at the method level AND at the class level. 

```java 
private String[] getRoles(final Method method) {
    if (method.isAnnotationPresent(Secure.class)) {
        return method.getAnnotation(Secure.class).roles();
    }

    if (method.getDeclaringClass().isAnnotationPresent(Secure.class)) {
        return method.getDeclaringClass().getAnnotation(Secure.class).roles();
    }

    return null;
}
```    

The last step to activate the interceptor is to declare it in the ``beans.xml`` file. Remember that it works
thanks to CDI and only for the beans managed by CDI. By default, interceptors are disabled.

```xml 
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://java.sun.com/xml/ns/javaee" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://jboss.org/schema/cdi/beans_1_0.xsd">
    <interceptors>
        <class>ch.demo.business.interceptors.SecurityInterceptor</class>       
    </interceptors>
</beans> 
```

#### How to populate the ThreadLocal variable with principal information

So far, we did intercept the method call and check whether the current principal associated with
the thread is authorised to access to a the intercepted method. The problem is that in servlet
containers the security is at web level and we would to propagate this information. To that end, we
will use the concept of [thread local](http://en.wikipedia.org/wiki/Thread-local_storage)
variables.

The following class contains a thread-local variable `` principalHolder`` that holds the current
(thread-local) principal. This assumes that a thread is processed by one and only one thread from A
to Z. This is the case for servlet engines but not always for application servers. In the latter
case, the vendor-specific security mechanism is much better as it takes things such as clustering
into account. The method `` setPrincipal`` is invoked at the beginning of a new request when it is
authenticated and authorised at the web level. ``removePrincipal`` cleans up the Thread-Local value 
at the end of the request processing.

```java 
public final class SecurityContext implements Serializable {

    private static InheritableThreadLocal<MyPrincipal> principalHolder = 
                new InheritableThreadLocal<MyPrincipal>();

    private SecurityContext() { }

    public static MyPrincipal getPrincipal() {
        if (principalHolder.get() == null) {
            principalHolder.set(new MyPrincipal(null, null));
        }
        return (MyPrincipal) principalHolder.get();
    }

    public static void setPrincipal(final MyPrincipal principal) {
        principalHolder.set(principal);
    }

    public static void removePrincipal() {
        principalHolder.remove();
    }
}
```

The last step is to populate the principal and its associated roles to the thread-local container
when the request is initialized. For that purpose, we use a request listener that is invoked when a new
request has been submit and just before its destruction. On initialization, the method 
``getMyPrincipal`` extracts the principal from the request ``request.getUserPrincipal()``. The
resulting principal is put in the thread-local by ``SecurityContext.setPrincipal``.

``getMyPrincipal`` also goes through the list of roles and adds the authorised roles using ``request.isUserInRole`` from ``HttpServletRequest``.

```java 
public class SecurityListener implements ServletRequestListener {

    private static List<String> roles = null;

    @Override
    public void requestInitialized(final ServletRequestEvent sre) {
        SecurityContext.setPrincipal(getMyPrincipal((HttpServletRequest) sre.getServletRequest()));
    }

    @Override
    public void requestDestroyed(final ServletRequestEvent sre) {
        SecurityContext.removePrincipal();
    }

    public static MyPrincipal getMyPrincipal(final HttpServletRequest request) {
        if (roles == null) {
            roles = getSecurityRoles(request.getServletContext());
        }

        Principal principal = (Principal) request.getUserPrincipal();
        List<String> currentRoles = new ArrayList<String>();
        for (String role : roles) {
            if (request.isUserInRole(role)) {
                currentRoles.add(role);
            }
        }
        return new MyPrincipal(principal, currentRoles);
    }

    public static synchronized List<String> getSecurityRoles(final ServletContext ctx) {
        ...
    }
}
```

The next method is not very elegant but this is the only way that have found to extract the role
list that is required by the current web application. It parses the ``web.xml``, looking for 
``role-name`` elements that describe a role.

```java 
public static synchronized List<String> getSecurityRoles(final ServletContext ctx) {
    List<String> r = new ArrayList<String>();
    InputStream is = ctx.getResourceAsStream("/WEB-INF/web.xml");

    DocumentBuilderFactory dbFactory = DocumentBuilderFactory.newInstance();
    dbFactory.setNamespaceAware(true);
    try {
        DocumentBuilder dBuilder = dbFactory.newDocumentBuilder();
        Document doc = dBuilder.parse(is);
        doc.getDocumentElement().normalize();

        NodeList elements = doc.getElementsByTagName("role-name");
        for (int i = 0; i < elements.getLength(); i++) {
            r.add(elements.item(i).getTextContent().trim());
        }
    } catch (Exception e) {
        new IllegalAccessException(e.getMessage());
    }
    return r;
}
```

Finally, the listener is enabled by putting the following lines in ``web.xml``.
```xml
<listener>
    <listener-class>ch.demo.web.SecurityListener</listener-class>
</listener>
```

Whenever there is not redirect (that triggers a new request) after login or there is a manual call to JAAS, the
principal must be manually set into thread-local right after the login. This is because if the
security is checked right after authentication and the listener has already been called (at the beginning of the
request) it has been set to an invalid principal.

```java
request.login(user, password);
SecurityContext.setPrincipal(SecurityListener.getMyPrincipal(request));
```

##Conclusion
In this post, we have seen how to write a JEE6 interceptor to authorise access to service level methods in servlet containers. The examples used in this blog are to be found in the [JEE-6-Demo](http://code.google.com/p/jee6-demo/) project on [Google code hosting](http://code.google.com).