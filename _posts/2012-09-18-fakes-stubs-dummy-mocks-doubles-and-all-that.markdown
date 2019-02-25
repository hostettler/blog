---
layout: post
title: "Fakes, Stubs, Dummy, Mocks, Doubles and all that..."
date: 2014-05-18 10:58
comments: true
categories:
published: true 
---

In this post, I  look at the different kind of objects used for test purposes. By this, I mean objects that are used to make a test running. This article focuses on component testing, a.k.a. unit testing (I do not like the term unit testing because it is too often misunderstood with the technology behind, e.g., JUnit, testNG).
Although there already exists a great number of resources on that subject, it was very difficult to me to understand the differences between the different kinds of test objects. This is partly due to the fact that different authors use different terms for the same object and the same term for different objects [1]. To be as didactic as possible, I also chose to add some blocks of code. Please note, that these blocks are only here for the sake of clarity. This is not the way I would recommend to do stubbing, faking, and mocking. Consider  [Mockito](https://code.google.com/p/mockito/) and PowerMockito for that. These are amazing tools to that purpose. They deserve a post on their own to discuss good practices.
This post is in no way an exhaustive state of the art, I only tried to select the terms that, in my opinion, are clear and are consensual enough. To that end I used a number of sources that can be found in the bibliography section.
   
Here are the main reasons to use different objects during the test phase and in production:

- Performances: the actual object contains slow algorithms and heavy calculation that may impair the test performances. A test should always be fast to not discourage regular run and therefore to identify problems as soon as possible. The worst case being the one in which the developer must deploy and run the entire application to test a single use case.

- States: sometimes the constellation under test happens rarely. This is for instances that occur with a low probability  such as race conditions, network failure, etc..

- Non-deterministic: this is the case of components that have interactions with the real-world such as sensors.

- The actual object does not exist: for instance, another team is working on it and is not yet ready.

- To instrument the actual dependency: for instance to spy the calls of the CUT to one of its dependencies.


## Doubles objects
Test double is the generic term that groups all the categories of objects that are used to fulfill one or several of the previous requirements.
The term comes has been coined by Gerard Meszaros in [2]
In rough terms, test doubles look like the actual object they double. They satisfy, to different extends the original interface and propose a sub-set of the behaviors that is expected by the specification. This helps to isolate the problem and reduce the double implementation to the strict minimum. 

There exists different kind of test doubles for different purposes. The have in common that they can be use instead of the actual component without breaking the contract syntactically. 

The next figure describes a simple test setup that do not use test doubles. To test the Component Under Test (CUT), the following test setup uses its actual dependencies (another component). This setup phase is trivial as there is nothing to do. The exercise phase calls the CUT with the proper parameters (direct inputs) that in turn calls it dependency (indirect outputs). Another Component returns its result to the CUT (indirect inputs) that uses it to complete the work and then finally returns the overall result (direct outputs). The terms "direct inputs", "indirect outputs", and so on come from [2].

{% include image.html url="/figures/TestSetup.png" description="Overview of a test setup" %}


Now let us say that "AnotherComponent" is either too complex, not already implemented or has a non-deterministic behavior. In those cases, it is easier to use another implementation of "AnotherComponent" that behaves exactly has expected for a specific scenario.

Hereafter, a simple example to illustrate the rest of the post. The class ``CUTImpl`` that realizes the contract ``CUT`` implements  the component under test. The CUT uses a component that realizes the interface ``AnotherComponent``. 
For the sake of clarity, the following example injects the dependencies through the constructor.
To improve loose coupling, it is possible to rely on dependency injection.
  
``` java
package ch.demo.business.service;

public class CUTImpl implements CUT {
	
	AnotherComponent component;

    public CUTImpl(AnotherComponent c) {
        this.component = c;
    }

	@Override
	public String doBusiness(String param, Integer delta) {
		return component.inc(Integer.valueOf(param)).toString();
	}
}
```

``` java
package ch.demo.business.service;

public interface CUT {
	public String doBusiness(String string, Integer delta);
}
```

``` java
package ch.demo.business.service;

public interface AnotherComponent {
	Integer inc(Integer param);
}
```

``` java
package ch.demo.business.service;

public class AnotherComponentImpl implements AnotherComponent {

    public Integer inc(Integer param) {
        if (param == null) {
            throw new IllegalArgumentException("Param must be not null!");
        } else if (param == Integer.MAX_INTEGER) {
            throw new IllegalStateException("Incrementing MAX_INTEGER will result in overflow!");
        } else {
            return param + 1;
        }
    }
}
```

The following test uses real implementations of the the different components. 

``` java
package ch.demo.business.service;

public class CUTTest {

    public void testInc() {
        Assert.assertEquals("inc(3) != 4", 4, new CUTImpl(new AnotherComponentImpl()).inc(3, 1));
    }

}
```

The question is what if:

1. ``AnotherComponentImpl`` is not ready yet 
2. ``AnotherComponentImpl`` depends itself on external services or specific hardware resources
3. ``AnotherComponentImpl`` has non-deterministic behaviors.


## Dummy objects

Dummy objects are meant to satisfy compile-time check and runtime execution. Dummies do not take part to the test scenario.
Some method signatures of the CUT may require objects as parameters. If neither the test nor the CUT care about these objects, we may choose to pass in a Dummy Object. This can be a null reference, an empty object or a constant. Dummy objects are passed around (to dependencies for instance) but never actually used. Usually they are just used to fill parameter lists. They are meant to replace input/output parameters of the components that the CUT interacts with. 

In the current example, the parameter ``delta`` of the ``doBusiness`` method can be set to ``null`` or any ``Integer`` value without interfering with the test. Of course, this might be different for another test.
 
``` java
package ch.demo.business.service;

public class CUTTest {

    public void testInc() {
        Assert.assertEquals("inc(3) != 4", 4, new CUTImpl(new AnotherComponentImpl()).inc(3, null));
    }

}
```

## Stub objects

Stub objects provide simple answers to the CUT invocations. It does answer to scenarii that are not foreseen by the current test. In other terms it is a simplified fake object. Stub objects may trigger paths in the CUT that would otherwise not been executed.

The next figure presents a test that relies on a test stub. First, the test case setups a stub object. This object responds to the expected CUT invokation in order to enact a given scenario. This is very useful to check indirect inputs with seldom values.

{% include image.html url="/figures/TestSTUB.png" description="Test setup that uses a test stub" %}

Back to the example, the following program illustrates how to use a stub to check specific indirect inputs.
This stub shows that the CUT relies on the fact that ``AnotherComponent`` does not return ``null``, as it
would otherwise raise a ``NullPointerException``.


``` java
package ch.demo.business.service;

public class AnotherComponentStub implements AnotherComponent {

    public Integer inc(Integer param) {
        return null;
    }

}
```

``` java
package ch.demo.business.service;

public class CUTTest {

    public void testIncWhenAnotherComponentReturnsNull() {
        //Without any modification of the CUT implement, this would raise an exception
        Assert.assertEquals("inc(3) != 4", 4, new CUTImpl(new AnotherComponentStub()).inc(3, 1));
    }

}
```

## Fake objects
Fake objects have working implementations, but  they may simplify some behaviors. This makes them not suitable for prime time. The idea is that the object actually displays some real behavior but not everything. While a Fake Object is typically built specifically for testing, it is not used as either a control point or an observation point by the test. The most common reasons for using fake objects is that the real component is not available yet, is too slow or cannot be used during tests because of side effects.

{% include image.html url="/figures/TestFAKE.png" description="Test setup that uses a fake" %}

The following fake simulates most of the behaviors except for the limits (MAX_INTEGER, null, etc...)

``` java
package ch.demo.business.service;

public class AnotherComponentFake implements AnotherComponent {

    public Integer inc(Integer param) {
        return param + 1;
    }

}
```

As the fake covers many scenarios, it can be used to test the general behavior of the CUT.
``` java
package ch.demo.business.service;

public class CUTTest {

    public void testIncWhenAnotherComponentIsFake() {
        CUT cut = new CUTImpl(new AnotherComponentFake());
        Assert.assertEquals("inc(3) != 4", 4, cut.inc(3, 1));
        Assert.assertEquals("inc(123) != 124", 124, cut.inc(123, 1));
    }
}
```

# Mock objects
Partially implements the interface and provides a way to verify that the calls to the mock objects validate the specification.
Mock objects are pre-programmed with expectations that form a specification of the calls they are expected to receive.
In fact mocks are a certain kind of stub or fake. However, the additional feature mock objects offer on top of acting as simple stubs or fakes is that they provide a flexible way to specify more directly how your function under test should actually operate. In this sense they also act as a kind of recording device: They keep track of which of the mock object's methods are called, with what kind of parameters, and how many times. 

Whenever the assertions are made on the fake object and not the CUT, then it is a mock.


{% include image.html url="/figures/TestMOCK.png" description="Test setup that uses a mock" %}

The following example uses [Mockito](https://code.google.com/p/mockito/) to provide easy Mocking. Note that the last assertion ``Mockito.verify`` checks whether the mock was called with a given parameters. In other words, we check that the CUT did not filter the input parameter.

``` java
package ch.demo.business.service;

public class CUTTest {

	@Mock
	AnotherComponent ac;
	
	@InjectMocks
	CUT cut = new CUTImpl();

    public void testIncWhenAnotherComponentReturnsNull() {
		Mockito.when(ac.inc(Integer.MAX_INTEGER)).thenReturn(Integer.MAX_INTEGER + 1);
		Mockito.when(ac.inc(3)).thenReturn(3);
		Mockito.when(ac.inc(123)).thenReturn(124);
		
        Assert.assertEquals("inc(Integer.MIN_INTEGER) != Integer.MIN_INTEGER + 1", 
                Integer.MIN_INTEGER + 1, cut.inc(Integer.MIN_INTEGER, 1));
        Assert.assertEquals("inc(3) != 4", 4, cut.inc(3, 1));
        Assert.assertEquals("inc(123) != 124", 124, cut.inc(123, 1));
        
        //Verifies that the method inc of AnotherComponent was called with parameter Integer.MAX_INTEGER
        Mockito.verify(ac).inc(Matchers.eq(Integer.MAX_INTEGER));
        //Verifies that the inc method has been called three times.
        Mockito.verify(ac, Mockito.times(3)).inc(anyInt());
    }
}
```

## Test Spy
According to Meszaros [2], a test spy is basically a recorder that is able to save the interactions between the CUT and the spy for later verifications.

{% include image.html url="/figures/TestSPY.png" description="Test setup that uses a test spy" %}
On the other hand, [Mockito](https://code.google.com/p/mockito/) considers that a spy is an real implementation in which you change only some specific behaviors. Instead of specifying every behavior one by one, you take an existing object that does most of it and you only change very specific behaviors.

## Conclusion
To sum up:

*   A dummy is just there to enable compilation and is not supposed to be part of the test.
*   A fake is a partial implementation that can be used either in a component test or in a deployed setting.
*   A mock is a partial implementation that enables asserting on the component interactions.
*   A spy is either a recorder for later use or a proxy on a real implementation that is used to override some specific behaviors.


## Bibliography

- [1] Martin Fowler [Mock aren't stubs](http://martinfowler.com/articles/mocksArentStubs.html)
- [2] Meszaros, Gerard (2007). xUnit Test Patterns: Refactoring Test Code. Addison-Wesley. ISBN 978-0-13-149505-0.
- [3] [Friends you can depend on](https://storage.googleapis.com/gtb/TotT-2008-06-12.pdf)
- [4] [Wikipedia Mock Object](http://en.wikipedia.org/wiki/Mock_object)
- [5] [Wikipedia Test Doubles](http://en.wikipedia.org/wiki/Test_double)
- [6] [Wikipedia Fakes](http://en.wikipedia.org/wiki/Fake_object)
- [7] [Wikipedia Stubs](http://en.wikipedia.org/wiki/Test_stubs)
