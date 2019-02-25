---
layout: post
title: "Testing JPA 2.0 entities in Java SE"
date: 2012-03-19 13:40
comments: true
categories:  [JPA 2.0, Java EE 6, JEE6, Java, Test, Java SE, JUnit, Derby, Java DB, Integration Test]
keywords:  [JPA 2.0, Java EE 6, JEE6, Java, Test, Java SE, JUnit, Derby, Java DB, Integration Test]
published: true
description: "In this article, we look how to test JPA 2.0 entities in Java SE. That is outside of the container." 
---

Testing JPA components in Java SE has been greatly simplified since JEE6. 
Nevertheless, it is still not as simple as testing other components. 
Although there exist some frameworks such as [Arquillian](http://www.jboss.org/arquillian) that solve that problematic, 
it is sometimes to much of hammer to test a couple of entities and simple services. 
Here is a post in which I digest some other posts and my own experience of how to test JPA components. The following example is part
of a Demo project I use to teach the JEE stack that is located on Google Code Hosting and [JEE-6-Demo](http://code.google.com/p/jee6-demo/)
This project relies on [Maven 3](http://maven.apache.org/) for build and dependency management. It does not use EJBs (at least for now) and thus it can be deployed in a Servlet engine.
Please note, that to me the kind of tests we are doing here are not unit tests (though we use JUnit ). Actually, as a database is required, 
these  are integration tests. 

Let us assume that we want to test that the following model has been properly annotated and can be persisted according to a given DB schema:
``` java  

// Get the entity manager for the tests.
@Entity
@Table(name = "STUDENTS")
public class Student implements Serializable {
    private static final long serialVersionUID = -6146935825517747043L;

    @Id
    @Column(name = "ID")
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long mId;

    @Column(name = "LAST_NAME", length = 35)
    private String mLastName;

    @Column(name = "FIRST_NAME", nullable = false, length = 35)
    private String mFirstName;

    @Column(name = "BIRTH_DATE", nullable = false)
    @Temporal(TemporalType.DATE)
    private Date mBirthDate;

}
```
with the following persistence unit:
```
<?xml version="1.0" encoding="UTF-8"?>
<persistence version="1.0"
    xmlns="http://java.sun.com/xml/ns/persistence">
    <persistence-unit name="JEE6Demo-Persistence"
        transaction-type="RESOURCE_LOCAL">
        <provider>org.eclipse.persistence.jpa.PersistenceProvider</provider>
        <class>ch.demo.dom.Student</class>
        <properties>
            <property name="eclipselink.target-database" value="MYSQL" />
            <property name="javax.persistence.jdbc.driver" value="com.mysql.jdbc.Driver" />
            <property name="javax.persistence.jdbc.url" value="jdbc:mysql://localhost:3306/Students_DB" />
            <property name="javax.persistence.jdbc.user" value="root" />
            <property name="javax.persistence.jdbc.password" value="" />           
            <property name="eclipselink.ddl-generation" value="none" />
            <property name="eclipselink.logging.level" value="INFO" />
        </properties>
    </persistence-unit>
</persistence>
```

We use [Eclipse-link](http://www.eclipse.org/eclipselink/) as it is the reference implementation for JPA 2.0
```xml  
	<repositories>
		<repository>
			<id>EclipseLink Repo</id>
			<name>EclipseLink Repository</name>
			<url>http://download.eclipse.org/rt/eclipselink/maven.repo</url>
		</repository>
	</repositories>

	<dependencies>
		<dependency>
			<groupId>org.eclipse.persistence</groupId>
			<artifactId>eclipselink</artifactId>
			<version>2.0.0</version>
			<scope>compile</scope>
		</dependency>
		<dependency>
			<groupId>org.eclipse.persistence</groupId>
			<artifactId>javax.persistence</artifactId>
			<version>2.0.0</version>
		</dependency>
	</dependencies>
```


Since JEE 6, it is easy to start the JPA manager in a unit test. Let us look at the following listing. The first two lines initialize the persistence manager given a persistence unit. The name of the persistence unit must matched the declared persistence unit in `META-INF/persistence`. After what, the try block gets a transaction and starts it. The object student is persisted and the transaction is committed. If anything goes wrong, the transaction is roll-backed. Finally, the entity manager and the factory are closed.
``` java  
// Get the entity manager for the tests.
EntityManagerFactory emf = Persistence.createEntityManagerFactory("JEE6Demo-Persistence");
EntityManager em = emf.createEntityManager();
try {
   //Get a new transaction
   EntityTransaction trx = em.getTransaction();

   //Start the transaction
   trx.begin();
   //Persist the object in the DB
   em.persist(student);
   //Commit and end the transaction
   trx.commit();
} catch (RuntimeException e) {
   if (trx != null && trx.isActive()) {
      trx.rollback();
   }
   throw e;
} finally {
   //Close the manager
   em.close();
   emf.close();
}
```

There are several open issues problems with this code:

1. We do want to use a local in memory database. Indeed, we do not want to mess up with the information stored on the Mysql database. Finally, we need the test to be as autonomous as possible. That is that it does not depend on an external server (e.g. app server, database server). 
2. JUnit does not guarantee the order in which the tests are ran. 
Therefore, we must initialize the manager before each and every test of the fixture. Furthermore we must close it after the test to 
avoid session leaks.
3. During tests, we must make sure that the database is in an acceptable state before running each unit.

To limit the dependencies to other components and to keep the setting simple, we will use [Derby](http://db.apache.org/derby/) as a pure in 
memory database. That is no file produced during the database execution and thus no cleaning afterwards. Please note that is only used for testing purposes.

```xml  
<dependency>
	<groupId>org.apache.derby</groupId>
	<artifactId>derby</artifactId>
	<version>10.8.2.2</version>
	<scope>test</scope>
</dependency>	
<dependency>
	<groupId>org.apache.derby</groupId>
	<artifactId>derbyclient</artifactId>
	<version>10.8.2.2</version>
	<scope>test</scope>
</dependency>
```

The following ``persistence.xml`` file sets  ``DERBY`` as a target database language. Using Derby, selecting the embedded 
drivers in conjunction ``org.apache.derby.jdbc.EmbeddedDriver`` with a ``jdbc:derby:memory`` URL runs a database in memory without 
persisting anything to disk. This combination is important as it otherwise will store file on the disk.
The following will be used by ``Maven`` during the test because it overrides the one in `/main/resources/META-INF/``. This is an elegant way to use a different database setting for the unit tests. 
``` xml  
<?xml version="1.0" encoding="UTF-8"?>
<persistence version="1.0"
    xmlns="http://java.sun.com/xml/ns/persistence">
    <persistence-unit name="JEE6Demo-Persistence"
        transaction-type="RESOURCE_LOCAL">
        <provider>org.eclipse.persistence.jpa.PersistenceProvider</provider>
        <class>ch.demo.dom.Student</class>
        <properties>
            <property name="eclipselink.logging.level" value="FINE" />
			<property name="eclipselink.target-database" value="DERBY" />
            <property name="javax.persistence.jdbc.driver" value="org.apache.derby.jdbc.EmbeddedDriver" />
            <property name="javax.persistence.jdbc.url" value="jdbc:derby:memory:StudentsDB;create=true" />
            <property name="javax.persistence.jdbc.user" value="" />
            <property name="javax.persistence.jdbc.password" value="" />
            <property name="eclipselink.logging.level" value="INFO" />
        </properties>        
    </persistence-unit>
</persistence>
```


To solve the second issue, we must guarantee that each test fixture gets a fresh entity manager. JUnit 4+ executes whatever 
public static method annotated with ``@BeforeClass`` before firing up the class constructor. Similarly, methods annotated with ``@AfterClass``
are executed after all the tests and is therefore a good place for cleanup.
``` java  

/** The factory that produces entity manager. */
private static EntityManagerFactory mEmf;
/** The entity manager that persists and queries the DB. */
private static EntityManager mEntityManager;

@BeforeClass
public static void initTestFixture() throws Exception {
    // Get the entity manager for the tests.
    mEmf = Persistence.createEntityManagerFactory(mPersistenceUnit);
    mEntityManager = mEmf.createEntityManager();
	...
}

 /**
 * Cleans up the session.
 */
@AfterClass
public static void closeTestFixture() {
    mEntityManager.close();
    mEmf.close();
}
```

The problem with the pure in-memory approach is that schema has to recreated for each run. One solution would be to rely on the ``Eclipse-link`` DDL creation. I do not really like this solution as it usually not accepted by the customers that tend to want to have control
of their schema. Thus, to have a test setting that is as close as possible to the production setting we need something to install the schema
by running a SQL file. There are many solutions out there:

- Manually parsing the SQL file. I do not like this solution as it becomes rapidly intractable when the file contains comments and transactions.
- Using tools such as [Liquibase](http://www.liquibase.org/). Nice solution but to me it was to much of a hammer to that specific case.
- Using an API that does it for me.

I propose the last approach by using ``Derby Tools`` and in particular the API of ``ij``.
```xml  
<dependency>
    <groupId>org.apache.derby</groupId>
    <artifactId>derbytools</artifactId>
    <version>10.8.2.2</version>
</dependency>
```	

This allows to use ``ij`` directly in the test to parse the DDL file. This is shown in the following listing.
First, it gets the underlying JDBC connection and then it run the file located in ``test/resources/sql/``.

```java 
@BeforeClass
public static void initTestFixture() throws Exception {
    // Get the entity manager for the tests.
    mEmf = Persistence.createEntityManagerFactory(mPersistenceUnit);
    mEntityManager = mEmf.createEntityManager();

    Connection connection = ((EntityManagerImpl) (mEntityManager
            .getDelegate())).getServerSession().getAccessor()
            .getConnection();

    ij.runScript(connection,
            AbstractDBTest.class.getResourceAsStream("sql/studentSchema.ddl"),
            "UTF-8", System.out, "UTF-8");

}
```

The third item asks for a way to properly initialize the data prior to the test. Remember that your tests should not rely on the success of
previous ones. To that end, we cleanup the data prior any test execution. A rather easy way to do that is to use 
[DBUnit](http://www.dbunit.org/).

```xml  
	<dependency>
		<groupId>org.dbunit</groupId>
		<artifactId>dbunit</artifactId>
		<version>2.4.8</version>
		<scope>test</scope>
	</dependency>
```	
It loads a file named `/test/resources/students-datasets.xml`` from the test resources.
```java  
    @BeforeClass
    public static void initTestFixture() throws Exception {
    // Get the entity manager for the tests.
    mEmf = Persistence.createEntityManagerFactory(mPersistenceUnit);
    mEntityManager = mEmf.createEntityManager();

    Connection connection = ((EntityManagerImpl) (mEntityManager
            .getDelegate())).getServerSession().getAccessor()
            .getConnection();

    ij.runScript(connection,
            AbstractDBTest.class.getResourceAsStream("sql/studentSchema.ddl"),
            "UTF-8", System.out, "UTF-8");
        
        mDBUnitConnection = new DatabaseConnection(connection);
        //Loads the data set from a file named students-datasets.xml
        mDataset = new FlatXmlDataSetBuilder().build(Thread.currentThread()
                .getContextClassLoader()
                .getResourceAsStream("students-datasets.xml"));

        ...
    }
```

After what, clean test data are inserted for each and every test of the fixture:

```java  
    @Before
    public void initTest() throws Exception {
        //Clean the data from previous test and insert new data test.
        DatabaseOperation.CLEAN_INSERT.execute(mDBUnitConnection, mDataset);
    }
```

In this post, I've showed how to test the persistence of an JPA 2.0 entity using JUnit 4. The test setting runs in memory without external server  and enforces a clean database state prior running the tests.

To support my teaching activities on Java EE 6, I've setup a project on Google Code Hosting that contains all the presented sources and much more. Please give me feedbacks on how to improve that demo project and in particular its
testability.




