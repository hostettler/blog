---
layout: post
title: "Using Selenium for web integration testing"
date: 2012-04-16 18:57
comments: true
categories: [Integration testing, test automation, Web application, Java EE6, JEE6, Selenium, JUnit]
keywords: [Integration testing, test automation, Web application, Java EE6, JEE6, Selenium, JUnit]
published: true
---

Following up on my previous posts ([here](/blog/2012/04/09/embedded-jee-web-application-integration-testing-using-tomcat-7/) 
and [here](/blog/2012/04/05/programmatically-build-web-archives-using-shrinkwrap/) about integration testing
of JEE web applications, I will present how to write smoke tests using Selenium. The idea is to
detect whether important regressions have been introduced. This is especially useful to detect
configuration and navigation problems. I've split the test in three parts:
 
1.  starting and stopping the container. Embedded Tomcat 7 in our case. This is described 
[here](/blog/2012/04/09/embedded-jee-web-application-integration-testing-using-tomcat-7);
2.  building the WAR file to test. To that end, you can look at this 
[post](/blog/2012/04/05/programmatically-build-web-archives-using-shrinkwrap) that describes how to
    build such a test archive using Shrinkwrap;
3.  finally, we must build the test case. This is the goal of this post.

Testing a use case requires to be able to:

1.  get a page at a given URL;
2.  input test data into the web elements;
3.  assert whether the server answers as expected.

Selenium provides an easy API to fetch URLs 
As these are smoke tests, using the ``HtmlUnitDriver`` instead of a specific browser is sufficient.
We do not want to detect minor display problems but rather important functional problems.

Selenium has many different drivers for many different browser. The problem is that it opens the
browser in a new window. I prefer to have the continuous integration running headless.

The following dependencies are required to run the tests.

```xml 
<dependency>
    <groupId>org.seleniumhq.selenium</groupId>
    <artifactId>selenium-java</artifactId>
    <version>2.20.0</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>net.sourceforge.htmlunit</groupId>
    <artifactId>htmlunit</artifactId>
    <version>2.9</version>
    <scope>test</scope>
</dependency>
```

First, let's build a new headless driver. This object simulates the browser. It manages the HTTP part as
well as parsing the HTML and the javascript code that is returned.
 
```java
WebDriver driver = new HtmlUnitDriver();
```

The next step is to ask the driver to load the home page that we want to test. For
convenience, in my previous posts I defined a method that returns the current Tomcat port and the
application context.
 
```java
driver.get("http://localhost:" + getTomcatPort() + "/" + getApplicationId());
```

Let's assume that the home page redirects to a login page that has two text boxes (respectively
called ``username`` and password). The following snippet uses XPATH expressions to locate web elements. 
First, it locates a HTML element of type ``input`` that has an
attribute  ``id`` set to ``username``. If this web element is not present, the driver will raise an exception.
This allows to easily detect whether a widget has been accidentally removed.

``` java 
WebElement username = driver.findElement(By.xpath("//input[contains(@id,'username')]"));
```	

Now we can interact with the previous element. For instance, to simulate a user that types in its username (``admin``). 
```java
username.sendKeys("admin");
```	

After that, Let's do the same for the web element called ``password``.
```java
WebElement password = driver.findElement(By.xpath("//input[contains(@id,'password')]"));
password.sendKeys("admin");
```	

Finally, we locate a button called ``login``and we simulate a click on it. 

```java
WebElement login = driver.findElement(By.xpath("//button[contains(@id,'login')]"));
login.click();
```	

On successful login, our example redirects to a page that contains a list of students.
The following assertion checks that the string ``Student list`` exists somewhere is the returned
HTML source code.

```java
Assert.assertTrue(driver.getPageSource().contains("Student list"));
```


With this approach, it is also very easy to do trivial security testing. For instance, let's check that
the application redirects (or forwards) the user to the login page upon unsuccessful login.
```java
WebElement username = driver.findElement(By.xpath("//input[contains(@id,'username')]"));
username.sendKeys("foo");
WebElement password = driver.findElement(By.xpath("//input[contains(@id,'password')]"));
password.sendKeys("foo");
WebElement login = driver.findElement(By.xpath("//button[contains(@id,'login')]"));
login.click();
Assert.assertTrue(driver.getPageSource().contains("username"));
Assert.assertTrue(driver.getPageSource().contains("password"));
Assert.assertTrue(driver.getPageSource().contains("login"));
```


## Conclusion
In this post, we have seen how to write smoke integration tests with Selenium. This is especially
useful to detect whether an important functionality has been hindered. For instance, a change in the
navigation or a unexpected change of the status (availability, visibility) of one the buttons or web elements.
I recommend implementing the three or four major scenarios with this technique. This is usually enough to
detect major problems. The examples used in this blog are to be found in the [JEE-6-Demo](http://code.google.com/p/jee6-demo/) project on [Google code hosting](http://code.google.com).