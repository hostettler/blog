---
layout: post
title: "Context Dependency Injection and the Rich Object Model"
date: 2012-12-05 06:59
categories: [CDI, Java EE6, JEE6, EJB 3.1, JPA 2.0, Multi-tenancy, Multi-site, Entity Manager]
keywords: [CDI, Java EE6, JEE6, EJB 3.1, JPA 2.0, Multi-tenancy, Multi-site, Entity Manager]
published: true
description: "In this post, I describe a way to get closer to a rich object model using the EJB 3.1 specification and more precisely the CDI 1.0 specification."
---


[Rich object model] vs [anemic object model] is long running debate. While the latter encourages to use simple and stupid objects with little or no business in them, the rich object model advocates for a clean object design with inheritance, polymorphism and so on.
The anemic object model is very popular among JEE partitioners because, in the past, the specification did not provide any mean to invoke services in business objects. Therefore, the anemic pattern uses so called "managers" that maintain references to other "managers". A direct benefit is the clear separation of concerns between the different kind of objects. Basically, it splits processing and data. As this is anti-object oriented, the abstract design of such system is often very different from the actual implementation. 

## The portfolio example

As example, let us take a portfolio that contains a set of financial position. A financial position can be either a set of stock, or an amount in a given currency. To evaluate the actual portfolio value, we go through the positions and for each of them we ask the current quote for stock to the service ``QuoteService`` or the current value of a given currency to the ``CurrencyService``.
The next figure presents the "ideal" design.

{% include image.html url="/figures/portfolio-business.png" description="An object oriented class diagram of the Portfolio management component." %}

To achieve this, one need to access services from within business objects. Since EJB 3.1,  Context and Dependency Injection (CDI) provides such a mechanism via the ``@Inject`` annotation. The only requirement is that the object that requires the service as well as the service to inject are so called "managed beans". The trick is that not all objects are meant to be managed. Furthermore, having managed lists of object is very tricky to say the least. Fortunately, the EJB 3.1 and more specifically the CDI 1.0 specification provide a way to solve this.
In CDI, the main component is the bean manager. This manager keeps track of the beans to inject via @Inject and other means. Instead of relying on annotations to provide injection, it is possible to use the good old Service Locator pattern. CDI 1.0 exposes the bean manager on JNDI with the name ``java:comp/BeanManager``.

```java
public class ServiceLocator {

   @SuppressWarnings("unchecked")
   public static <T> T getInstance(final Class<T> type) {
      T result = null;
      try {

         //Access to the current context.
         InitialContext ctx = new InitialContext();
         //Resolve the bean manager
         BeanManager manager = (BeanManager) ctx.lookup("java:comp/BeanManager");
         //Retrieve all beans of that type
         Set<Bean<?>> beans = manager.getBeans(type);
         Bean<T> bean = (Bean<T>) manager.resolve(beans);
         if (bean != null) {
            CreationalContext<T> context = manager
                  .createCreationalContext(bean);
            if (context != null) {
               result = (T) manager.getReference(bean, type, context);
            }
         }
      } catch (NamingException e) {
         throw new RuntimeException(e);
      }
      return result;
   }
}
```

The client code is very simple. It consists in calling the ``ServiceLocator``with the desired interface.
For the sake of clarity, I did not show the ServiceLocator that takes a qualifier in addition to the interface. To add this feature, look at the ``getBeans(Type beanType, Annotation... qualifiers)`` method.

```java

public class StockPos {

   private Long qty;
   private String stockId;

   Double evaluate() {
      StockQuoteService sqs = ServiceLocator.getInstance(StockQuoteService.class);
      return qty * sqs.getQuoteValue(stockId);
   }

}
```

Similarly, here is the code of the ``CurrencyPos`` object.

```java

public class CurrencyPos {

   private Double amount;
   private String currencyId;

   Double evaluate() {
      CurrencyQuoteService sqs = ServiceLocator.getInstance(CurrencyQuoteService.class);
      return amount * sqs.getQuoteValue(stockId);
   }

}
```

## Some thoughts on the Demeter law

Let me be clear, I do **not** recommend this approach everywhere. It is very important to not mix the objects responsabilities. Furthermore,
in order to respect the Demeter law, a business must **not** directly call something outside of the current component. Calls to other components are always to be done through so-called consumers to have clear components boundaries.
For instance, putting to much intelligence in JPA entities that can be detached and serialized may cause problems on the client side.

## Conclusion
In this post, I showed a solution to consume services that are exposed via the CDI BeanManager. These services can be pure POJOs or EJBs.
Nevertheless, this approach must be used with great care as it can blur the components boundaries and responsabilities.
