---
layout: post
title: "Communication between Java Enterprise Applications"
date: 2014-12-26 22:51:48 +0100
comments: true
published : true
categories:  [EARS, JEE, RMI]
keywords:  [EARS, JEE, RMI]
---

Recently I came across the following problem : How to propagate information from one enterprise application to another in a transparent manner? Transparent meaning without changing the API, that is without adding transversal information to the services' parameters. The typical use case is to propagate information such as the language, applicative security roles information (not the JAAS role), or the session-id. Moreover, I would like the information to be "request scoped" and to be automatically cleaned up at the end of the request. This is important to avoid memory leak and to enforce isolation for security reasons. Let me add that I currently work with JEE6 on WebSphere 8.5.5.

I did some research among blogs and forums and I found the following solutions:

1. Passing information in thread-local. This consists in putting information in a `Map` stored in `ThreadLocal`. Although this solution is very simple to implement, information is only transmitted inside the current thread. This means that `@Asynchronous` calls will not get access to the information. Similarly, it is not transmitted through RMI calls that spread on several VMs, as it is usually the case on distributed applications.

2. Using the JNDI. This solution does not suffer of the above limitations as it is distributed by nature but scoping must be implemented on top of the existing JNDI implementation. I think that it may be possible to implement something like a custom CDI scope but it seems rather complex.

3. TransactionSynchronizationRegistry (TSR). This alternative is well documented [here](http://www.adam-bien.com/roller/abien/entry/how_to_pass_context_in). This solution works on a JEE application servers. It looks great at the first sight but it does not support any use case in which there is different transactions (or no transaction) involved. This invalidates any information sharing before a transaction as started, when a transaction has been suspended, or when a new transaction is started. Again, I can imagine that it would possible to propagate the content using interceptors but it is too much plumber code to me.

4. Work Area Service (WAS). This is basically the IBM implementation of solution 2 with scoping. Documentation is clear and it seems easy to implement. Of course, the main drawback is that it is vendor-specific. IBM started a [JSR](https://jcp.org/en/jsr/detail?id=149) long time ago but it was dropped.


Let us now enumerate several criteria to make a decision about which way to go:

1. Supports of asynchronous calls : during a request it may be necessary to dispatch the processing among several threads and I would like the shared information to be accessible by any threads involved in this request.  

2. Is (automatically) "request scoped" : if the shared information is not automatically collected at the end the request we may end up with memory leaks. Manual collection is never a good option.

3. Supports Remote Calls : for a given request, we may end up calling several services (EJBs) on others servers and I would like to have an automatic propagation of the information among the clusters nodes.

4. Performance : to be useful, the information sharing must be ubiquitous and therefore it must cheap in terms of resources.

5. Vendor Independence : as far as possible an application must rely on known and portable APIs such as JEE. Locking the application to a specific vendor is, in my opinions, is essentially a problem for maintenance. Migrating from one application server to another only happens rarely.

<table >
<tr > 
<th >Solution</th> <th    >Async</th> <th    >Scope</th> <th    >RMI</th> <th    >Vendor Indep.</th> </tr>
<tr> <td >Thread-Local</td    > <td    >X</td> <td    >OK</td> <td    >X</td> <td    >OK</td> </tr>
<tr> <td >JNDI</td> <td    >OK</td> <td    >X</td> <td    >OK</td> <td    >OK</td> </tr>
<tr> <td >TSR</td> <td    >X</td> <td    >OK</td> <td    >OK</td> <td    >OK</td> </tr>
<tr> <td >WAS</td> <td    >OK</td> <td    >OK</td> <td    >OK</td> <td    >X</td> </tr>
</table>


<br/>


As you can see there is no silver bullet here. I went for the vendor specific solution. It can be nicely encapsulated to isolate the dependency to vendor specific code. Furthermore, several servers have similar mechanism and it can be therefore adapted. Here is why it was not possible in my setup to use the other alternatives:
 
- The Thread-local solution is not acceptable because it does not support Remote Method Invocation on several virtual machines.
- The JNDI solution requires to implement the scoping mechanism. This can be tricky and it is definitely not my area of expertise.
- The TransactionSynchronizationRegistry is JEE compliant but it requires huge machinery to support asynchronous calls as well as transaction suspension and re-creation (`REQUIRES_NEW`, `NOT_SUPPORTED`, `NEVER`). Basically, it does not work if there is not one and only one transaction throughout the request.


## Bibliography
[1] Adam Bien. HOW TO PASS CONTEXT IN STANDARD WAY - WITHOUT THREADLOCAL. http://www.adam-bien.com/roller/abien/entry/how_to_pass_context_in

[2] Adam Bien. HOW TO PASS CONTEXT BETWEEN LAYERS WITH THREADLOCAL AND EJB 3.(1). http://www.adam-bien.com/roller/abien/entry/how_to_pass_context_with

[3] IBM. Work area partition service. http://www-01.ibm.com/support/knowledgecenter/SSEQTJ_8.0.0/com.ibm.websphere.nd.multiplatform.doc/info/ae/workarea/concepts/cwa_partition.html