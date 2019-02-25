---
layout: post
title: "How to simulate CDI scopes and injection in Java SE."
date: 2012-04-02 21:37
comments: true
categories: [CDI, Java EE6, JEE6, Weld, JUnit, Injection, Scopes]
keywords: [CDI, Java EE6, JEE6, Weld, JUnit, Injection, Scopes]
published: true
description: "In this blog, we look at how to simulate CDI scopes and injection to test managed beans in Java SE."
---

Unit testing managed beans is difficult outside of a container. Managed beans heavily rely on the notions of scopes and injection that do not exist outside of a container. In JEE6, both are handled by CDI (Context Dependency Injection). 
[Arquillian](http://www.jboss.org/arquillian.html) is a powerful solution to this problem. Nevertheless, sometimes for technical or even for political reasons, it is not possible to add a new component to the existing stack.
While searching for alternatives, I came across several interesting articles ([here](http://www.jtips.info/index.php?title=WeldSE/Scopes), [here](http://objectopia.com/2011/05/29/weld-junit-4-runner/) and [here](http://danhaywood.com/2010/08/12/simulating-cdis-session-and-request-scope-in-a-j2se-app/)) that explain how to simulate such features in unit tests. 

This post aims at consolidating these articles for Weld 1.1.5 and JUnit 4.5. 
The following example is part of a Demo project that I use to teach the JEE stack that is located on Google Code Hosting and [JEE-6-Demo](http://code.google.com/p/jee6-demo/)

The following snippet presents a unit test of a managed bean. The bean to test (and all its dependencies) is injected in the test. As you can see the test is simple and straightforward.
The injected scopes (Conversation in this case) can be used during the test to setup a particular case.

```java  
@RunWith(WeldJUnit4Runner.class)
public class ManageStudentRegistrationTest {

	/** Service injected by the Weld container. */
	@Inject
	private ManageStudentRegistration mManageStudentRegistration;

	/** A conversation for the test. */
	@Inject
	private Conversation mConversation;

	@Test
	public void testPieChartCreation() {
		PieChartModel model = this.mManageStudentRegistration.getPieModel();
		Assert.assertNotNull(model);
		Assert.assertEquals(4, model.getData().size());
	}

	@Test
	public void toRegistrationTest() {
	    this.mConversation.begin("ConversationId");
		Assert.assertEquals("register", mManageStudentRegistration.toRegistration());
	}

}
```

The fist step is to enable CDI injection in unit tests. To that end, we extends the ``BlockJUnit4ClassRunner`` that is responsible for creating a Test case.
The constructor simply initializes the Weld container. Finally, we override the
test creation. Instead of directly invoking the constructor of the test class, we ask Weld to instantiate it (line 25). This will inject all dependencies into the test object. In our case, it will create  the manager bean and its dependencies.

```java
public class WeldJUnit4Runner extends BlockJUnit4ClassRunner {

    /** The test class to run. */
    private final Class<?> mKlass;
    /** Weld infrastructure. */
    private final Weld weld;
    /** The container itself. */
    private final WeldContainer container;

    /**
     * Runs the class passed as a parameter within the container.
     * @param klass to run
     * @throws InitializationError if anything goes wrong.
     */
    public WeldJUnit4Runner(final Class<Object> klass) throws InitializationError {
        super(klass);
        this.mKlass = klass;
        this.weld = new Weld();
        this.container = weld.initialize();
    }

    
    @Override
    protected Object createTest() throws Exception {
        final Object test = container.instance().select(mKlass).get();
        return test;
    }
}
```

Remember to declare a ``META-INF/beans.xml`` file in the test resources in order to provide
mock implementations. In this case, we enable the alternative ``StudentServiceMockImpl`` that is used by the managed bean that is under test.

```xml 
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://java.sun.com/xml/ns/javaee" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/beans_1_0.xsd">
    <alternatives>
        <class>ch.demo.business.service.mock.StudentServiceMockImpl</class>
    </alternatives>
</beans> 
```

Until there, the dependency injection does not manage the scopes that may be used in managed bean. To that end, we must add an [extension](http://docs.jboss.org/weld/reference/latest/en-US/html/extend.html) to Weld. Extensions are powerful mechanisms to tweak the container behavior. In this case, we want to add the missing scopes. This is done in the following snippet. It listens to the ``AfterDeploymentValidation`` event that occurs after the configuration as been validated but before context creation. Methods ``afterDeployment`` creates a map for each scope and associates it.
```java
package org.jboss.weld.manager; // required for visibility to BeanManagerImpl#getContexts()
...
/**
 * Taken from http://www.jtips.info/index.php?title=WeldSE/Scopes,
 * it simulates request and session scopes outside of an application server.
 */
public class WeldServletScopesSupportForSe implements Extension {
	
	/** {@inheritDoc} */
	public void afterDeployment(@Observes final AfterDeploymentValidation event, 
					final BeanManager beanManager) {
					
		Map<String, Object> sessionMap = new HashMap<String, Object>();
		activateContext(beanManager, SessionScoped.class, sessionMap);

		Map<String, Object> requestMap = new HashMap<String, Object>();
		activateContext(beanManager, RequestScoped.class, requestMap);

		activateContext(beanManager, ConversationScoped.class, 
				new MutableBoundRequest(requestMap, sessionMap));
	}

	/**
	 * Activates a context for a given manager.
	 * @param beanManager in which the context is activated
	 * @param cls the class that represents the scope
	 * @param storage in which to put the scoped values
	 * @param <S> the type of the storage
	 */
	private <S> void activateContext(final BeanManager beanManager,
				final Class<? extends Annotation> cls, final S storage) {
		BeanManagerImpl beanManagerImpl = (BeanManagerImpl) beanManager;
		@SuppressWarnings("unchecked")
		AbstractBoundContext<S> context = 
			(AbstractBoundContext<S>) beanManagerImpl.getContexts().get(cls).get(0);

		context.associate(storage);
		context.activate();
	}
}
```

To register and activate a CDI extension, a file that contains the extension class name must be present and named ``META-INF/services/javax.enterprise.inject.spi.Extension``.
```java  
org.jboss.weld.manager.WeldServletScopesSupportForSe
```

## Conclusion
Testing the JEE6 outside of the container is easier and easier. Long gone are the days of the EJB 2.1 untestability.
Nevertheless, not everything is simple and we still have to do some tricks to get it working. 