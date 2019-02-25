---
layout: post
title: "Multi-Tenancy with EJB 3.1 and JPA 2.0"
date: 2012-11-20 05:18
categories: [CDI, Java EE6, JEE6, EJB 3.1, JPA 2.0, Multi-tenancy, Multi-site, Entity Manager]
keywords: [CDI, Java EE6, JEE6, EJB 3.1, JPA 2.0, Multi-tenancy, Multi-site, Entity Manager]
published: true
description: "In this post, we look at how to simulate multi-tenancy with several datasources using JPA 2.0"
---

Multi-tenancy is a recurrent non functional requirement. Indeed, many important IT-systems are meant to be shared among multiple tenants. The data are often distributed over several databases or schemas. This, for different reasons:

- Security: The data belong to different customers and some level of isolation is required;
- Performances: Distributing the data over multiple systems may help to master performance issues;
- Legacy: Sometimes, old and new systems must cohabit for a (long) time;
- Maintenability: A database or a schema can be updated without putting the rest of the application at risk.

Although data are distributed, the application code should remain tenant agnostic. Furthermore, choosing between the different tenants is often made at runtime based on credentials (e.g. user Joe has access to customer AAAA while user Jane sees data of customer BBB). [Java EE 7 will address this problem and much more](/https://blogs.oracle.com/arungupta/entry/java_ee_7_key_features), but in the mean time here is the way that I use to address this problematic using EJB 3.1 and JPA 2.0

## Overall architecture

First, let me start with the overall architecture as described below.

{% include image.html url="/figures/multi-tenancy-architecture.png" description="Multi-tenancy architecture with serveral datasources " %}


In the above figure, the database is organized in schemas, with one application server datasource (DS) per schema and one persistence unit (PU) per datasource. 
It is also possible to use only one datasoure and to discriminate between schemas by setting the ``<property name="openjpa.jdbc.Schema" value="TenantX" />`` property for each persistence unit (PU). This sets the default schema for the PU.
Here is a ``persistence.xml`` file that provides one persistence unit per tenant. 

The following code has been tested for Open-JPA but there is nothing specific to this implementation outside of the ``<provider>``tag in the ``persistence.xml``file.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<persistence version="2.0"
   xmlns="http://java.sun.com/xml/ns/persistence" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
   xsi:schemaLocation="http://java.sun.com/xml/ns/persistence http://java.sun.com/xml/ns/persistence/persistence_2_0.xsd">

   <persistence-unit name="Tenant1" transaction-type="JTA">
      <provider>
         org.apache.openjpa.persistence.PersistenceProviderImpl
      </provider>
      <jta-data-source>jdbc/Tenant1_DS</jta-data-source>
      <exclude-unlisted-classes>false</exclude-unlisted-classes>
   </persistence-unit>


   <persistence-unit name="Tenant2" transaction-type="JTA">
      <provider>
         org.apache.openjpa.persistence.PersistenceProviderImpl
      </provider>
      <jta-data-source>jdbc/Tenant2_DS</jta-data-source>
      <exclude-unlisted-classes>false</exclude-unlisted-classes>
   </persistence-unit>

</persistence>
```

The basic idea is that, instead of using ``@PersistenceContext``, we inject our "own" multi tenant entity manager wraper.
Then, at runtime, the multi-tenant entity manager loads the persistence context that corresponds to the current user context from JNDI.
Please note that this only works for JTA-based persistence-units. Otherwise, the persistence context is not container-basd and therefore not exposed to JNDI. Moreover, without JTA, we loose container based transaction demarcation.


Let us first start with the client code. In other words, how to use the Multi-Tenant Entity manager.

## Client Code

Here is the client code. In order to preserve thread-safety and transactionality, Data access objects are EJBs (``@Stateless``, ``@Stateful``, ``@Singleton``). The presented solution uses an entity manager that is wrapped and then injected using ``@Inject`` or ``@EJB``.  Thread-safety, transactionnality and performances are guaranted by the ``EJB 3.1`` and ``JPA 2.0`` specification as explained in the section ``Thread-safety and Transactionality``. As shown below, the ``MultiTenancyWrapper`` delegates to a real entity manager and implements the ``EntityManager`` interface. Therefore, its use is very similar to a normal ``EntityManager`` injected via ``@PersistenceContext``.

```java
@Stateless
public class MyEJB implements MyEJBLocal {

   @Inject
   private MultiTenancyWrapper emWrapper;

   @TransactionAttribute(TransactionAttributeType.REQUIRED)
   public void doSomething() {
      emWrapper.findAll(...);
   }
}
```

## The Multi-Tenant EntityManager EJB

The ``MultiTenanEntityManagertWrapper`` simply wraps the entity manager that corresponds to the current user context. The trick is to configure it as an EJB in order to get the xml configuration feature via ``ejb-jar.xml``. Another alternative would be to use the ``@PersistenceContexts`` and ``@PersistenceContext`` annotations. The main drawback being that, for each new tenant, not only the ``persistence.xml`` and ``ejb-jar.xml`` must be changed but also the ``Java`` code.

The JNDI context that is linked to the current request is injected in the ``MultiTenantEntityManager`` using the ``@Resource``annotation.
As there is no creation of a new ``InitialContext`` the overhead is not significant. Actually, the ``@PersistentContext`` annotation does the exact same thing except that it is not specific to the user context. The ``MultiTenanEntityManagertWrapper`` implements the delegate pattern. This allows to use it (almost) transparently in client code.
The main difference being the use of ``@Inject`` or ``@EJB`` over ``@PersistenceContext`` in the client code.

Using the session context that is specific to the caller bean (and thus the caller request/session) enables transparent support for thread-safety, security and transactionality.

```java
package ejb;

import javax.persistence.EntityManager;

public interface MultiTenanEntityManagertWrapper extends EntityManager {

}
```

The method ``getMultiTenantEntityManager`` of the ``MultiTenanEntityManagertWrapperImpl`` extracts the ``EntityManager`` that corresponds to the current request from JNDI (we will see later how it has been put there). To that end, the method ``getMultiTenantEntityManager``first extracts the prinipal from the current EJB context (``SessionContext``). After what, the tenant that corresponds to the current user is used to obtain the JNDI name of the corresponding entity manager. ``MultiTenanEntityManagertWrapperImpl`` simple delegates every call to the this Request specific ``EntityManager``.
 
```java

@Stateless
public class MultiTenanEntityManagertWrapperImpl implements MultiTenanEntityManagertWrapper {
   
   private static final String JNDI_ENV = "java:comp/env/persistence/";

   @Resource
   SessionContext context;

   
   private EntityManager getMultiTenantEntityManager() {
      //Extract the name of the current user.
      Principal p = context.getCallerPrincipal();

      //Lookup the tenant name for the current user
      //This is application specific
      Users u = Users.getUser(p.getName());

      //Produces either TENANT1 or TENANT2      
      String tenantName = u.getSite().toString();

      String jndiName = new StringBuffer(JNDI_ENV).append(tenantName).toString();
      //Lookup the entity manager
      EntityManager manager = (EntityManager) context.lookup(jndiName);
      
      if (manager == null) {
         throw new RuntimeException("Tenant unknown");
      }
      return manager;
   }


   //The delegates
   @Override
   public void persist(Object entity) {
      getMultiTenantEntityManager().persist(entity);
   }


   @Override
   public <T> T merge(T entity) {
      return getMultiTenantEntityManager().merge(entity);
   }


   @Override
   public void remove(Object entity) {
      getMultiTenantEntityManager().remove(entity);
   }

        ...
}
```

Now let us see how to put the entity manager references in JNDI.
In order to avoid a lot of annotations (one per tenant) and therefore to be able to handle a huge number of tenans, I propose to use the ``ejb-jar.xml`` file to configure the EJB intead of the ``PersistenceContext`` annotation. The ``MultiTenantEntityWrapper``EJB is configured as a stateless EJB. Ther persistence contexts are simply exposed to JNDI with the following pattern: ``java:comp/env/persistence/TENANTX``. For more information please look at the EJB 3.1 specification chapter 16.11.1.

``<persistence-unit-name>Tenant1</persistence-unit-name>`` is the name of the PU as defined in the ``persistence.xml`` file. ``   <persistence-context-ref-name>persistence/TENANT1</persistence-context-ref-name>``defines the name of the entity manager that is exposed via JNDI.


```xml
<?xml version="1.0" encoding="UTF-8"?>
<ejb-jar xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://java.sun.com/xml/ns/javaee" xmlns:ejb="http://java.sun.com/xml/ns/javaee/ejb-jar_3_0.xsd" xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/ejb-jar_3_1.xsd" version="3.1">


   <enterprise-beans>
      <session>
         <ejb-name>MultiTenantEntityWrapper</ejb-name>   
         <ejb-class>ejb.MultiTenantWrapperImpl</ejb-class>
         <session-type>Stateless</session-type>

         <!-- Persistece contexts -->
         <persistence-context-ref>
            <description>Tenant 1</description>            
            <persistence-context-ref-name>persistence/TENANT1</persistence-context-ref-name>      
            <persistence-unit-name>Tenant1</persistence-unit-name>
            <persistence-context-type>Transaction</persistence-context-type>
         </persistence-context-ref>

         <persistence-context-ref>
            <description>Tenant 2</description>            
            <persistence-context-ref-name>persistence/TENANT2</persistence-context-ref-name>      
            <persistence-unit-name>Tenant2</persistence-unit-name>
            <persistence-context-type>Transaction</persistence-context-type>            
         </persistence-context-ref>
         
      </session>
   </enterprise-beans>

</ejb-jar>
```

## Thread-safety and Transactionality
As this is compliant with both the EJB 3.1 and JPA 2.0 specification, thread-safety and transactionnaly are guaranteed by the container. For more details please look at the EJB 3.1 specification
at chapters 16.10, 16.11 and the JPA 2.0 specification at chapter 7.6. Of course, the wrapper has to be an EJB in order to have access to the current JNDI context without having to create it.
Furthermore, because the ``EntityManager``is not ``per se`` thread-safe (JPA 2.0, chapter 7.2), the serialization of the invokations that is provided by the container for EJBs is essential the thread-safety aspect (EJB 3.1, chapter 4.10.13).


## Conclusion

In this post, I showed how to leverage EJB 3.1 and JPA 2.0 standard features to provide multi-tenancy. The presented approach is thead-safe, it preserves transactionaly and does not induce 
a significant overhead.
