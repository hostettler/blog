---
layout: post
title: "JEE6 Tutorial"
date: 2012-12-31 09:43
comments: true
categories: 
---

I hereby start a series of tutorial on the major (IMHO) JEE 6 features. As a lecturer, I teach the JEE stack at the University of Geneva. This tutorial represents most of the topics that I cover during the EJB, CDI, and JPA lessons.

There already exists a bunch of very complete tutorials and books on the subject. Let me cite the excellent [Oracle JEE tutorial] [1] and 
Antonio Goncalvez's books that are available both in [English] [2] and in [French] [3]. For advanced JEE developers, Adam Bien's [Real World Java EE Night Hacks] [7] is a must read.   

As the [EJB 3.1 specification] [5] is rather well written, I encourage people to read it or at least to look at specifics when needed.

To support this tutorial you can find a JEE6-Demo on GitHub. It contains a simple enterprise application that demonstrates many of the
aspects I discuss hereafter. The first step of this tutorial is to prepare the environment:

1) Install GlassFish and integrate it to your favorite development environment (eclipse in my case). You can get GlassFish from [here] [11]. Grab the zip version and install it by unzipping the archive.

2) Run GlassFish and check that it works. Locate the script named ``asadmin`` in the ``bin`` directory, start it and at the prompt, execute the command ``start-domain``. This starts the GlassFish instance.

```
localhost:bin ~$ ./asadmin
Use "exit" to exit and "help" for online help.
asadmin> start-domain
Waiting for domain1 to start ...
Successfully started the domain : domain1
domain  Location: ~/Applications/glassfish3/glassfish/domains/domain1
Log File: ~/Applications/glassfish3/glassfish/domains/domain1/logs/server.log
Admin Port: 4848
Command start-domain executed successfully.
asadmin> 
```
Now the server is started and it listens to the ports 8080 and 4848. The first being the application port while the second is the administrative console.

Start a browser and navigate to [``http://localhost:4848/common/index.jsf``](http://localhost:4848/common/index.jsf). Depending of the server configuration, you may have to log in.

3) Do a Git checkout of the jee6-demo project that is available [here](https://github.com/hostettler/JEE6-Demo).

4) Perform a ``mvn clean install`` at the project root

This compiles the project and builds the artifacts. It should end up in something like:
```
[INFO] ------------------------------------------------------------------------
[INFO] Reactor Summary:
[INFO] 
[INFO] jee6-demo ......................................... SUCCESS [0.480s]
[INFO] Service Layer ..................................... SUCCESS [3.650s]
[INFO] Presentation Layer ................................ SUCCESS [1.725s]
[INFO] Application ....................................... SUCCESS [0.663s]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 6.923s
[INFO] Finished at: Sun Jan 06 16:41:29 CET 2013
[INFO] Final Memory: 19M/81M
[INFO] --------------------------------------------------------------------
```

5) In the GlassFish installation directory, there is a ``javadb`` directory that contains, Derby, a Java-based database. Starts the network server:
```
>$ pwd
/~/Applications/glassfish3/javadb/bin
localhost:bin ~$ ./startNetworkServer
Mon Jan 07 06:02:54 CET 2013 : Security manager installed using the Basic server security policy.
Mon Jan 07 06:02:54 CET 2013 : Apache Derby Network Server - 10.8.1.2 - (1095077) started and ready to accept connections on port 1527
```
The database server listens to the port ``1527``.


Now that the database is started. It is time to populate the database with the database creation scripts as well as some data.
In the project, under the ejb components, there is a sql file that initializes the database structure as well as some data.

Derby provides a command line tool called ``ij`` to connect to the database and to execute sql scripts:

```
localhost:bin ~$ pwd
~/Applications/glassfish3/javadb/bin
localhost:bin ~$./ij ~/git/jee6demo/jee6-ejb/src/main/resources/sql/createStudentsDB_DERBY.sql
...
ij> QUIT;
```

Now the database is ready to use.


6) Create a datasource in GlassFish that points to the  database server that has been started previously:

First, let us create a connection pool:

```
asadmin> create-jdbc-connection-pool  --datasourceclassname=org.apache.derby.jdbc.ClientDataSource40 --restype=javax.sql.DataSource --property user=DEV:DatabaseName=StudentsDB:Password=DEV:ServerName=localhost:PortNumber=1527:create=true StudentsDB
Authentication failed with password from login store: ~/.asadminpass
Enter admin password for user "admin"> 
222JDBC connection pool StudentsDB created successfully.
Command create-jdbc-connection-pool executed successfully.
```

The connection pool provides a ... pool of connection that is managed by the application server. A major benefit being that the developer does not have to open and close connections.
The server opens n connections and allocate them on demand. It ensures that there will not be 1000 open connections to the database thus improving the performances.
Then, we create a datasource that uses the previous connection pool:

```
asadmin> create-jdbc-resource --connectionpoolid StudentsDB --enabled jdbc/StudentsDS
Authentication failed with password from login store: ~/.asadminpass
Enter admin password for user "admin"> 
JDBC resource jdbc/StudentsDS created successfully.
Command create-jdbc-resource executed successfully.
```

7) Configure the messaging resources.

The tutorial uses JMS (Java Messaging Service) to implement the publish/subscribe pattern. The following script builds a connection factory for JMS queues.
```
asadmin> create-jms-resource --restype javax.jms.QueueConnectionFactory --enabled  MyConnectionFactory
Authentication failed with password from login store: ~/.asadminpass
Enter admin password for user "admin"> 
Connector resource MyConnectionFactory created.
Command create-jms-resource executed successfully.
```

The following snippet declares the JMS queue. That is the message endpoint.

```
asadmin> create-jms-resource --restype javax.jms.Queue MyQueue
Authentication failed with password from login store: ~/.asadminpass
Enter admin password for user "admin"> 
Administered object MyQueue created.
Command create-jms-resource executed successfully.
```

8) Configure the security

Authentication and authorization is another major feature that is provided by the application server.
First let us create the realm with two groups user and manager.

```
asadmin> create-auth-realm --classname com.sun.enterprise.security.auth.realm.file.FileRealm  --property assign-groups=manager,user:file=jee6-tutorial-file-realm:jaas-context=fileRealm jee6-tutorial-file-realm
Authentication failed with password from login store: ~/.asadminpass
Enter admin password for user "admin"> 
Command create-auth-realm executed successfully.
```

Then, we add a user that belongs two groups: user and manager.
```
asadmin> create-file-user --authrealmname jee6-tutorial-file-realm --groups user:manager user1
Authentication failed with password from login store: ~/.asadminpass
Enter admin password for user "admin"> 
Enter the user password> 
Enter the user password again> 
Command create-file-user executed successfully.
```

The second user only belongs to the group user.
```
asadmin> create-file-user --authrealmname jee6-tutorial-file-realm --groups user user2
Authentication failed with password from login store: ~/.asadminpass
Enter admin password for user "admin"> 
Enter the user password> 
Enter the user password again> 
Command create-file-user executed successfully.
```


9) Deploy the application

```
asadmin> deploy ~/git/jee6demo/jee6-ear/target/jee6-demo-ear-1.0.0-SNAPSHOT.ear
Authentication failed with password from login store: ~/.asadminpass
Enter admin password for user "admin"> 
Application deployed with name jee6-demo-ear-1.0.0-SNAPSHOT.
Command deploy executed successfully.
```

10) Test the application

Once deployed, serve to [``http://localhost:8080/jee6-demo-web/facade/studentService/all``](http://localhost:8080/jee6-demo-web/facade/studentService/all) and you should get:
You must log yourself with one of the use that you created at step 8).

```xml
<studentsDto xmlns:ns2="http://ch.demo.app">
	<students>
		<id>0</id>
		<last_name>Doe</last_name>
		<firstName>John</firstName>
		<birthDate>1965-12-10T00:00:00+01:00</birthDate>
		<phoneNumber>
		<areaCode>0</areaCode>
		<countryCode>0</countryCode>
		<number>0</number>
		</phoneNumber>
		<address>
			<number>22</number>
			<street>wisteria street</street>
			<city>Downtown</city>
			<postalCode>34343</postalCode>
		</address>
	</students>
	<students>
		<id>1</id>
		<last_name>Doe</last_name>
		<firstName>Jane</firstName>
		<birthDate>1985-12-10T00:00:00+01:00</birthDate>
		<phoneNumber>
		<areaCode>0</areaCode>
		<countryCode>0</countryCode>
		<number>0</number>
		</phoneNumber>
		<address>
			<number>22</number>
			<street>wisteria street</street>
			<city>Downtown</city>
			<postalCode>34343</postalCode>
		</address>
	</students>
</studentsDto>
```


Now you are ready to follow me in the EJB world. 

The tutorial is organized in three parts. First, in this post, we will look at the EJB stack. In following posts, we will cover both the Context and Dependency Injection and the Java Persistence API.

## <a id="ejbs"></a>EJBs

The EJB API provides useful non-functional services to Java Enterprise Applications such as transactionality, security, pooling, and thread-safety.
Server side components that implement this API and wrap up business logic are called Enterprise Java Beans (EJBs). While the first versions were infamously known for the boilerplate code as well as their low testability,
EJB 3.x are rather easy to use thanks to a new design paradigms called [Convention over Configuration] [4]. The idea is to specify unconventional aspects rather than everything. This dramatically reduces the boilerplate code and improves readability.
 
Furthermore, the massive usage of annotations instead of XML descriptor greatly improves the productivity. In the sequel, I mostly rely on annotations. Nevertheless, every annotation has its counterpart in a configuration descriptor that is called ``ejb-jar.xml``. 

> In case there is two different value for the same property, remember that one from the ``ejb-jar.xml`` always overrides the annotation. 
 
Enterprise Java Beans are of three kinds:

- **Session Beans**: components whose execution is triggered by a client call. The call can be local (within the JVM) or remote (by another JVM).
- **Message Beans**: execution is triggered by a message and the business logic is processed asynchronously. Please note that, since JEE6 there are other (simpler) ways to process logic asynchronously. Nevertheless, Message Beans
remain very useful to implement bus and/or publish-subscribe patterns and to enforce loose coupling between the client and the server component.
- **Entity Beans**: I will not discuss entity beans as they have been replaced by the [JPA 2.0 specification] [6].


### <a id="sb"></a>Session Beans
As mentioned previously, session beans (merely) react on client invocation. Henceforth, the bean can either take the client session into account (thus it shares a state between multiple client invocations) or it can treat each call as unrelated.
In the first case, the EJB is called **Stateful** while in the latter it is called **Stateless**.

### <a id="slsb"></a>Stateless Session Beans
Let us start with a **Stateless** bean. The following Java class describes a Stateless EJB.
The most important part is the annotation ``@Stateless`` at the top of the class.
This marks the class as an EJB. When the container instantiates this class, it knows that
it is a so-called **Managed Bean**. By this, it means that the container manages the bean's lifecycle (e.g., instantiation, passivation). Furthermore, as an EJB, it has access to the container services such as security and transactionality.

The following snippet describes a Stateless Session bean that computes a grade distribution. The method is secured and only authorized to client that have the role ``user`` through the ``@RolesAllowed({ "user" })`` annotation.

``` java
@Stateless
public class StudentServiceJPAImpl implements StudentService {
...
    @Override
    @RolesAllowed({ "user" }) // This a role-based security.
    public final Integer[] getDistribution(final int n) {
    	numberOfAccess++;
        Integer[] grades = new Integer[n];

        for (Student s : this.getAll()) {
            grades[(s.getAvgGrade().intValue() - 1) / (TOTAL / n)]++;
        }
        return grades;
    }
 ...
}
```

Please note that the previous EJB implements an interface. An EJB can be local (invokable from within the container),
remote (invokable from another JVM) or both. In the previous case, it is a local interface and therefore the bean is only
invokable locally.

``` java
@Local
public interface StudentService extends Serializable {
	/**
	 * @return an array that contains the distribution of the grades. It
	 *         partitions the grades in "n" parts.
	 * @param n : number of parts
	 */
	int[] getDistribution(final int n);
}
```

Adding a remote invocation to the implementation amounts to add an interface that is annotated ``@Remote``.
Of course, the interface may be different and may exposes different services locally and remotely.

```java
@Remote
public interface StudentServiceRemote extends Serializable {

    /**
     * @param lastname
     *            of the student
     * @return the student with the given lastname.
     */
    Student getStudentByLastName(String lastname);

}
```

The implementation must be changed as follow to be remotely invokable:

``` java
@Stateless
public class StudentServiceJPAImpl implements StudentService, StudentServiceRemote {
...
```

To use this EJB, we need a client code. The easiest possible interaction is either via a JAX-RS service or via a servlet.
In the JEE6-Demo, under the JEE6-WEB project the JAX-RS facade is called `StudentFacade.java`. It implements a JAX-RS facade over the ``StudentService``.

The method ``getDistribution()`` uses the student service that is injected via ``@EJB`` in the 
facade. [Dependency injection] [10]  is now a first class citizen in the JEE6 stack. 

```java
@ApplicationScoped // This sets the scope to application level
@Path("/studentService")
public class StudentServiceFacade implements Serializable {
...
	@EJB
	StudentService studentService;

	@GET
	@Produces({ "application/xml", "application/json" })
	@Path("distribution")
	public Response getDistribution() {
		Integer[] distrib = studentService.getDistribution(10);
		return Response.ok(new DistributionDto(distrib)).build();
	}
}
```

The following snippet illustrates the injection of a Stateless EJB into a Servlet.

```
@WebServlet("/student")
public class StudentServlet extends HttpServlet {

	private static final long serialVersionUID = 1L;

	@EJB
	StudentService studentService;

	@Override
	protected void doGet(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
		response.getOutputStream()
				.println("There are #" + studentService.getAll().size() + " students in the database");
	}

	@Override
	protected void doPost(HttpServletRequest arg0, HttpServletResponse arg1) throws ServletException, IOException {
		this.doGet(arg0, arg1);
	}

}
```

As you can see, implementing EJBs (and invoking them) is rather easy. So far, we did not talk much about transactionality and security so you might wonder why using EJBs for such simple business services. The answer is pooling and thread-safety. As the container manages a pool of EJBs, it handles the invocation queue and you do not have to manager re-entrance and thread-safety.

> Remember that there is a **one to many relationship** between a stateless session bean and the clients. In other words, one instance may manage different clients. Hence, there must be not field that represent a client state in a **Stateless** EJB because there is no guarantee that a given client will get the same SLSB accross two invokations. Nevertheless, within a given invokation the client is guaranteed to be the only user.


Consider the following exercises.

> *Exercise 1:* Add a new method to the StudentService service that computes the standard deviation and use this method in the JAX-RS facade.

> *Exercise 2:* Add a counter that is incremented each and every time a method is invoked. Print out the thread-id (and if possible the session-id) and explain why the so-called stateless beans must be without state. Add a rest service to interface this method.

#### <a id="trx"></a>Transactionality

[Transactions] [8] are important concepts when dealing with enterprise applications. It is of vital importance that whenever something goes wrong, the transaction is roll-backed. This preserves the [ACID] [9] properties. Managing database transactions with JDBC is complex to say the least. Remember that we often have to manage transactions over several databases and even between databases and messaging queues. If the processing of a message leads to a database exception, we might want to put the message back in the queue.

Lucky for us, JEE 6 manages (almost) everything by itself. By default a bean is transactional and each method either reuse a running transaction or creates a new one if necessary.

>By default the transaction (or transaction context) is propagated to other business methods that are called from within a transactional method.

In JEE, there is two types of transaction management:

- **Bean Managed** : In this case the bean is annotated with ``@TransactionManagement(TransactionManagementType.BEAN)``. In this mode, the developer must manage the transaction himself.

- **Container Managed**: this is the default behavior and it corresponds to marking the bean ``@TransactionManagement(TransactionManagementType.CONTAINER)``. This mode is called **Container Managed Transaction Demarcation**. This idea is that every public method is transactional and is marked as ``@Required``. 

The following snippet shows how to use bean demarcation:
```java
@MessageDriven(mappedName = "MyQueue")
@TransactionManagement(TransactionManagementType.BEAN)
public class StudentRegistrationService implements MessageListener {

	@PersistenceContext
	EntityManager em;

	...

	public void onMessage(Message inMessage) {
		TextMessage msg = null;

		em.getTransaction().begin();

		try {
			...
			msg = (TextMessage) inMessage;
			logger.info("MESSAGE BEAN: Message received: " + msg.getText());
			logger.info(Thread.currentThread().getName());
			...
		} catch (JMSException e) {
			em.getTransaction().rollback();
			...
		}
		em.getTransaction().commit();
	}
}
```

and now the exact same class using container demarcation:
```java
@MessageDriven(mappedName = "MyQueue")
public class StudentRegistrationService implements MessageListener {

	@PersistenceContext
	EntityManager em;

	...

	public void onMessage(Message inMessage) {
		TextMessage msg = null;
		try {
			...
			msg = (TextMessage) inMessage;
			logger.info("MESSAGE BEAN: Message received: " + msg.getText());
			logger.info(Thread.currentThread().getName());
			...
		} catch (JMSException e) {
			em.getTransaction().rollback();
			...
		}
	}
}
```

Because the previous snippet relies on the default convention, it is equivalent to:
```java
@MessageDriven(mappedName = "MyQueue")
@TransactionManagement(TransactionManagementType.CONTAINER)
@TransactionAttribute(TransactionAttributeType.REQUIRED)
public class StudentRegistrationService implements MessageListener {

	@Inject // Let's ignore this for the moment
	private transient Logger logger;

	@PersistenceContext
	EntityManager em;

	@Resource // Let's ignore this for the moment
	private MessageDrivenContext mdbContext;

	public void onMessage(Message inMessage) {
		TextMessage msg = null;
		try {
			...
			msg = (TextMessage) inMessage;
			logger.info("MESSAGE BEAN: Message received: " + msg.getText());
			logger.info(Thread.currentThread().getName());
			...
		} catch (JMSException e) {
			em.getTransaction().rollback();
			...
		}
	}
}
```
In other terms, it uses container managed transaction demarcation ``@TransactionManagement(TransactionManagementType.CONTAINER)``.
Moreover, each and every public method either reuses an existing transaction or creates a new one if none has been started ``@TransactionAttribute(TransactionAttributeType.REQUIRED)``

The ``@TransactionAttribute`` annotation can also be put directly on the method. Valid options are:

- ``REQUIRED``: if there is an existing transaction then it reuses it, otherwise it creates a new one.
- ``REQUIRES_NEW``: it always creates a new transaction 
- ``NEVER``: if there is an existing transaction then it fails
- ``MANDATORY``: if there is an existing transaction then it reuses it, otherwise it fails
- ``NOT_SUPPORTED``: If there is a transaction then it suspends it and it does not propagate the transaction to other business methods, otherwise it does nothing.
- ``SUPPORTS``: If a transaction exists then it works as for ``REQUIRED``, otherwise it works as for ``NOT_SUPPORTED``.

Consider the following exercises.

> *Exercise 3:* Test the different types of transaction attribute and observe their respective behavior.

> *Exercise 4:* Are the snippets 1 and 2 semantically equivalent?

> *Exercise 5:* Mix the two types of transaction management.

#### <a id="secu"></a>Security

Another great feature of EJBs is their ability for declarative authorization and the integration in the JAAS framework. JAAS is the Java Authentication and Authorization Service
It provides unified services to access user directories for user authentication and their authorization. As the client is already authenticated, at the EJB level, only authorization matters. 

After authentication, a principal (e.g., user name, role) is set in the context. This principal is used for authorization.

The main annotation is ``@RolesAllowed`` followed by a list of authorized roles. As an example, look at the method ``getDistribution`` of ``StudentServiceJPAImpl.java``. The method is only allowed for client that belongs to the group/role ``user``.

Sometimes, a method of an EJB is not executed in the context of an authenticated user or the authenticated user has not enough credentials. If it is still important to run the method, you can use the ``@RunAs`` annotation. This basically overrides the current security context (and the associated principal) similarly to ``sudo`` for UNIX.

In any EJB, it is possible to get the current client session and therefore the principal:

```java
@Resource 
SessionContext ctx;

public void doSomething() {
	// obtain the principal. 
	Principal principal = ctx.getCallerPrincipal();
	// obtain the name. 
	String name = callerPrincipal.getName();
	// Is the caller an admin?
	ctx.isCallerInRole("admin")
}
```

> *Exercise 6:* Change the method getDistribution to only allow client that belong to the ``admin`` group. Log as a user and as an admin, check the servlet as well as the JAX-RS service. What do you observe?

> *Exercise 7:* Add a new Role to the application, and use it. Warning, some configurations are vendor specific.

> *Exercise 8:* Create a service that uses ``@RunAs`` to upgrade the credentials of a standard user in order to call another service that requires the admin level. 

#### <a id="callbacks"></a> Callbacks and Interceptors

Interceptors are used to enhance the business method invocations and the beans' lifecycle events. Interceptors implements the AOP paradigm.

Interceptors are of two kinds:
- Lifecycle event interceptors such as ``@PostConstruct``, ``@PostActivate``, and ``@PrePassivate``, and ``@PreDestroy`` enable to add logic when the bean is created or destroyed.
- Call-based interceptors that relies on the ``@AroundInvoke`` annotation. 

In the first case, adding the annotation before the method declaration is enough. It will then be called when the lifecycle event occurs.
In the latter case, in addition to the annotation the developer must configure the ``ejb-jar.xml`` file.

Here is a simple interceptor that prints out the time consumed by a method call:

```java
@AroundInvoke
public Object intercept(InvocationContext ctx) throws Exception
{
   long start = System.currentTimeMillis();
      
   try {
		Object value = context.proceed();
   } finally {
   	 	StringBuilder str = new StringBuilder("******  ");
		str.append(context.getMethod().getName());
		str.append(" took :");
		str.append(System.currentTimeMillis() - start);
		str.append(" ms  ******");

		mLogger.info(str.toString());
   }
}
```

In addition to the interceptor declaration, it must be activated by mean of either an annotation on the target EJB or via the ``ejb-jar.xml`` file. If several interceptors are declared, then the order of declaration is the order of execution: the last is the most nested interceptor. 

The following example presents the annotation based declaration.

```java
@Stateless
@Interceptor({PerformanceEJBInterceptor.class})
public final class StudentServiceJPAImpl implements StudentService, StudentServiceRemote {
``` 

In the following snippet, the interceptor is applied to all methods whose name is like ``get*`` and EJB's name is like ``Student*``. It is also possible to specify the parameters' type in case of overloading.

``` xml
<assembly-descriptor>
 <!-- Default interceptor that will apply to all methods for all beans in deployment -->
 <interceptor-binding>
      <ejb-name>Student*</ejb-name>
      <method>
           <method-name>get*</method-name>
      </method>
      <interceptor-class>ch.demo.business.interceptors.PerformanceEJBInterceptor</interceptor-class>      
   </interceptor-binding>
   ...
</assembly-descriptor>
```

> *Exercise 9:* Add an interceptor that for all public methods of ``StudentServiceJPAImpl.java`` to increments an overall (common to all clients) usage counter using the annotation based approach.
 
> *Exercise 10:* Do the same, but using the ``ejb-jar.xml`.

#### <a id="timers"></a> Timers

Some services are not directly linked to a client session. This is the case for batch processes that must run every x hours. To that end, EJB 3.1 provides the annotation ``@Schedule`` to specify how often a service method must be called. This is similar to the CRON  feature on UNIX systems.

The following example runs the method ``doSomethingUseful`` every 30 minutes.
```java
	@Schedule(minute="*/30", hour="*")
    public void doSomethingUseful() {
        ...
    }  
```

> *Exercise 11:*  Add a scheduler that runs a method every minutes to print out the number of students in the database.

> *Exercise 12:*  ``@Schedule`` is often used in conjunction with ``@RunAs``. Why?

#### <a id="async"></a> Asynchronous calls

Asynchronous calling means that the control is returned to the caller before the process has been completed. For instance, a batch job that loads 1000 customers description from one database, processes them and finally save them in another database. From a performances point of view, it is better to split the process in x batches and then to wait until every thing is finished. Firstly, if the asynchronous call is made to another EJB, it is possible to leverage the pool to have parallel treatments without managing multi-threading and race conditions. Secondly, as the different call may use different database connections, it will limit the database bottleneck.

To do this, JEE provides the ``@Asynchronous`` annotation. Any method that returns either nothing (``void``) or an instance of type ``Future<T>`` can be run asynchronously.

The ``Future`` interface exposes the method ``get`` that blocks until the job is over.
The following class, exposes the method ``processJob`` that runs asynchronously.

The methods ``processJobs`` starts 10 times the method processJob and put the resulting ``Future<String>`` in an array. Then it iterates of the pool and wait until all calls have returned.

``` java
@Stateless
public class BatchProcessor {

public void processJobs() {
	Future<String>[] pool = new Future<String>[10];
	for (int i = 0; i < 10; i++) {
		pool[i] = processJob("Job #" + i);
	}

	for (int i = 0; i < 10; i++) {
		pool[i].get();
	}	
}


@Asynchronous
public Future<String> processJob(String jobName) {

    try {
        Thread.sleep(10000);
    } catch (InterruptedException e) {
        Thread.interrupted();
        throw new IllegalStateException(e);
    }
    return new AsyncResult<String>(jobName);

}

```

> *Exercise 13:* Create an EJB that invoke an asynchronous method that runs 100 times. The asynchronous method should wait a random time between 5 and 10 seconds  before returning a the actual waiting time. Collect all the waiting times and compute the average.

### Singleton Beans

Singleton beans (``@Singleton``) are special kind of EJBS that exist only in one instance in the container. This is for instance useful to initialize the application. Thus, ``@Singleton`` can be associated with ``@Startup`` to start the EJB during container's startup.

As there is only one instance, it means the bean is meant to be shared. Therefore, it is important to specify the locking policy. There are two types of locking:
- Container managed concurrency management ``@ConcurrencyManagement(ConcurrencyManagementType.CONTAINER)``. This is the default behavior and it is highly recommended to stick to it. Annotating a method (or the class) with ``@Lock(LockType.READ)`` means that the method can be safely access concurrently. ``@Lock(LockType.WRITE)`` requires the calls to the method to be serialized. The methods are ``@Lock(LockType.WRITE)`` by default.
- Bean managed concurrency management ``@ConcurrencyManagement(ConcurrencyManagementType.BEAN)``. This requires the developer to use synchronized and volatile to achieve good concurrency.


> By default the class is marked as ``@Lock(LockType.WRITE)``, thus EVERY CALL is synchronized. This is probably not the expected behavior (at least most of the time) and produces a huge bottleneck. Make sure to set the proper lock policy on the class and to only put ``@Lock(LockType.WRITE)`` where needed.

The following bean initialize a shared variable during container startup and lock the access to the modification to only one thread (client) at a time.

```java
@Singleton
@Startup
@Lock(LockType.READ)
public class StatusBean {
  private String status;

  @PostConstruct
  void init {
    this.status = "Ready";
  }

  public String getStatus() {
    return this.status;
  }
  
  @Lock(LockType.WRITE)
  public void setStatus(String new Status) {
    this.status = newStatus;
  }

  @Lock(LockType.WRITE)
  @AccessTimeout(value=360000)
  public void doTediousOperation {
    ...
  }
}
```


> *Exercise 14:* Write a singleton that reads the postal code from the database during startup, update them every 30 minutes. Take care of the concurrency aspects.


### <a id="sfsb"></a>Stateful Session Beans

Unlike stateless session beans, stateful session beans maintain a state across several client invocation. This is because there is a one to one relationship between a client and a stateful session bean.

The following code snippet describes a stateful session bean that maintains a counter across several invocations.
A stateful session bean is annotated with ``@Stateful``. As it is stateful, it maintains a session state and it is therefore important to set a timeout. Otherwise, the number of open session will raise and it will consume the server memory.
The timeout is defined using the ``@StatefulTimeout`` annotation.

> Because there is no instance sharing among several client, Stateful session beans are more resource-demanding than stateless session beans. Therefore, they should be used with great care.

```java
@Stateful
@StatefulTimeout(unit = TimeUnit.MINUTES, value = 30)
public class StudentStatisticsServiceImpl implements StudentStatisticsService {

	private Long numberOfAccess = 0l;

	@Override
	public String getStatistics() {
		return "The student service has been invoked #" + numberOfAccess
				+ " in this session";
	}

	public void count() {
		numberOfAccess++;
	}
}
```

As a stateful session bean is not per se linked to an HTTP session, the application client code must ensure to remember which instance of the EJB is dedicated to which client.
When used in conjunction with a Servlet, Stateful beans are often put in the HTTP session for further reuse.

```java
@Override
protected void doGet(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
		StudentStatisticsService statistics = (StudentStatisticsService) request.getSession().getAttribute(
				StudentStatisticsService.class.toString());

		if (statistics == null) {
			try {
				InitialContext ctx = new InitialContext();
				statistics = (StudentStatisticsService) ctx
						.lookup("java:global/JEE6-EAR/JEE6-EJB-0.0.1-SNAPSHOT/StudentStatisticsServiceImpl");
				request.getSession().setAttribute(StudentStatisticsService.class.toString(), statistics);
			} catch (NamingException e) {
				throw new RuntimeException(e);
			}

		}

		statistics.count();

		response.getOutputStream().println("There are #" + studentService.getAll().size() + " students in the database");
		response.getOutputStream().println(statistics.getStatistics());
		response.getOutputStream().println(studentService.getStatistics());
}
```

Either the EJB is looked-up with JNDI and then put into the HTTP session or, when using CDI, the link is made based on the caller scope.
The class ``StudentServiceFacade.java`` describes how the injection automatically links the correct instance of a stateful session bean to a given based on the ``@SessionScoped`` annotation.

```java
@SessionScoped
@Path("/studentService")
public class StudentServiceFacade implements Serializable {

	private static final long serialVersionUID = 1318211294294344900L;

	@EJB
	StudentService studentService;

	@EJB
	StudentStatisticsService studentServiceStatistics;


	@GET
	@Produces({ "application/xml", "application/json" })
	@Path("session")
	public Response getSessionStatistics() {
		return Response.ok(studentServiceStatistics.getStatistics()).build();
	}
	
	...
}
```

Consider the following exercise.

> *Exercise 15:* Show that two sessions share the same overall count but not the same per session count.

As the container does not share the stateful instances, it might run out of memory and therefore it will try to save the state of the bean on disk (or database). This operation is called **Passivation**. Afterwards, when the client comes back and requires its stateful instance, the container loads it from the disk. This operation is called **Activation**.

Some operations such as initializing/cleaning File, a socket connection, or a database connection can be made before activation and/or after activation using the following 
``@PrePassivate`` and the ``@PostActivate`` annotations.

Consider the following exercises:

> *Exercise 16:* Stop the server and observe that the bean has been passivated.

> *Exercise 17:* Start the server and observe that the bean has been activated.

Whenever the user logs out and thus invalidate the HTTP Session, you might want to clean up the EJB state and release the resources. To achieve this, the client must call a method annotated with ``@Remove``. As soon as the client call is over the container can garbage the bean. The idea is simply to get ahead of the timeout.

> *Exercise 18:* Implements two methods with ``@Remove`` with two different behaviors. Check that both will discard the current instance. Hint: you can use ``@PreDestroy`` to check the bean's destruction.


## Message Driven Beans

Message driven beans (MDB) are very useful for asynchronous process and for decoupling the client from the processor. MDB implement a typical publish/subscribe architecture.

There are two kinds of objects, a MDB can listen to:

- **Topics**: pure publish/subscribe semantics. A new message will be processed by every listeners.
- **Queue**: more of a load balancer, once the message is consumed by one of the listeners, it
is removed from the queue.


Although, there is, in JEE6, another mechanism for asynchronous processing, MDBs remain the best alternative if you do not want the client to be aware of the asynchronous logic and processor.

The idea is that there is a queue (or a topic) in between. Therefore, the client as no knowledge whatsoever of the message processor contract or semantics.

The first step is to configure a Queue Connection Factory as well as a Queue in the application server.


The following servlet gets  the connection factory as well as the queue injected. As always with dependency injection, the same result can be achieved using JNDI lookups.
For each request to the servlet 100 messages are generated and put in the Queue called ``MyQueue``.

```java
@WebServlet("/registration")
public class RegistrationServlet extends HttpServlet {

	...
		
	@Resource(mappedName = "MyConnectionFactory")
	private ConnectionFactory connectionFactory;

	@Resource(mappedName = "MyQueue")
	private Queue queue;

	@Override
	protected void doGet(HttpServletRequest request, HttpServletResponse response) throws 	ServletException, IOException {

		try {
			Connection connection;
			connection = connectionFactory.createConnection();

			Session session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
			MessageProducer messageProducer = session.createProducer(queue);

			TextMessage message = session.createTextMessage();

			for (int i = 0; i < 100; i++) {
				message.setText("This is message " + (i + 1));
				messageProducer.send(message);
			}
		} catch (JMSException e) {
			throw new RuntimeException(e);
		}
	}

}
```

The following MDB listens to ``MyQueue`` (thanks to the annotation ``@MessageDriven(mappedName = "MyQueue")`` and because it implements the ``MessageListener`` interface it takes a message when available.


```java
@MessageDriven(mappedName = "MyQueue")
public class StudentRegistrationService implements MessageListener {
    ...

	@Resource
	private MessageDrivenContext mdbContext;

	public void onMessage(Message inMessage) {
		TextMessage msg = null;

		try {
			if (inMessage instanceof TextMessage) {
				msg = (TextMessage) inMessage;
				logger.info("MESSAGE BEAN: Message received: " + msg.getText());
			} else {
				logger.warn("Message of wrong type: "
						+ inMessage.getClass().getName());
			}
		} catch (JMSException e) {
			mdbContext.setRollbackOnly();
			throw new RuntimeException(e);
		}
	}
}
```

Here is a sample output. As you can see, the messages are processed according to their input order but not necessarily by the same EJB (worker).

```
INFO: Sending message: This is message 1
INFO: Sending message: This is message 2
INFO: Sending message: This is message 3
INFO: MESSAGE BEAN: Message received: This is message 1
INFO: MESSAGE BEAN: Message received: This is message 2
INFO: Sending message: This is message 5
INFO: Sending message: This is message 6
INFO: MESSAGE BEAN: Message received: This is message 4
INFO: MESSAGE BEAN: Message received: This is message 5
...
INFO: MESSAGE BEAN: Message received: This is message 100
```

Consider the following exercises:

> *Exercise 19:* Compare Message Driven Beans and Session Beans with asynchronous calls.

> *Exercise 20:* Print out the thread-id. What do you observe?

> *Exercise 21:* Create another MDB that consumes the same queue. What do you observe?

> *Exercise 22:* Do the same (2 consumers) but using a ``Topic`` instead of a ``Queue``.



[1]: http://docs.oracle.com/javaee/6/tutorial/doc/ "Oracle's official JEE6 tutorial"
[2]: http://www.amazon.com/Beginning-GlassFish-Experts-Voice-Technology/dp/143022889X/ref=sr_1_1?ie=UTF8&qid=1356945347&sr=8-1&keywords=JEE6+antonio+goncalves "Antonio's Book in English"
[3]: http://www.amazon.com/Java-EE6-GlassFish-Antonio-Goncalves/dp/2744024236/ref=sr_1_fkmr2_1?s=books&ie=UTF8&qid=1356945440&sr=1-1-fkmr2&keywords=JEE6+et+glassfish "Antonio's Book in French"
[4]: http://en.wikipedia.org/wiki/Convention_over_configuration "Convention over Configuration on Wikipedia"
[5]: http://jcp.org/aboutJava/communityprocess/final/jsr318/index.html "EJB 3.1 specification"
[6]: http://jcp.org/aboutJava/communityprocess/final/jsr317/index.html "JPA 2.0 specification"
[7]: http://press.adam-bien.com/real-world-java-ee-night-hacks-dissecting-the-business-tier.htm "Real World Java EE Night Hacks--Dissecting the Business Tier"
[8]: http://en.wikipedia.org/wiki/Database_transaction "Transactions on wikipedia"
[9]: http://en.wikipedia.org/wiki/ACID "ACID in wikipedia"
[10]: http://en.wikipedia.org/wiki/Dependency_injection "Dependency Injection on Wikipedia"
[11]: http://glassfish.java.net/downloads/3.1.2.2-final.html "Glassfish Download Page"