---
layout: post
title: "Testing levels"
date: 2014-04-18 10:52
comments: true
categories:  [Unit tests, Acceptance Tests, Tests Levels, Integration Tests, Verification, Validation]
keywords:  [Unit tests, Acceptance Tests, Tests Levels, Integration Tests, Verification, Validation]
published: true 
---
## Introduction

In this post, I would like to discuss number of definitions around the testing activity. Having these definitions in mind helps to organize this crucial activity. In a previous post, I discussed the difference between [verification and validation](). If the difference is not clear to you, please have a look at it prior reading this post.

Let me start with the definition of what is testing. Software testing helps to measure the quality of a software in terms of defects. It is crucial to understand that 
"testing shows the presence, not the absence of bugs" [1].
 

This comes from the fact that exhaustive testing is not possible  due to a phenomena called the state space explosion [2]. The idea is that doing exhaustive testing would require a structure in memory that remembers all the tested states of the system. A state of the system being the concatenation of its variables. For instance, let us take a program that has two variables:
- an integer (4 bytes = 32 bits)
- an array of ASCII characters of length 10 (10 bytes =  80 bits)
The number of states to explore is  2^112 ~ 5x10^33 states (remember that the number of atoms in the observable universe is 10^80) and the required amount of memory would be 7.2x10^22 Terabytes. Although many optimizations can be brought to a brute force approach [1,2], the problem remains huge. Therefore, exhaustive testing is not an option.

Another important point about defects is to understand from where they originate.
A (software) defect originates in a human **mistake** (e.g., a misunderstanding) that produces a **fault** (i.e., a defect, a bug). Under certain circumstances, the faulty code will end up doing something unexpected with respect to the user requirements. This is called a **failure**.

To sum up, there is the following causality chain : Mistake --> Fault --> Failure.  
 
This demonstrates that testing is not only a matter of detecting the failure but that it can be done earlier. Of course the earlier the defect is detected the cheaper is it to address it. For instance, informing the developer about the business may avoid a mistake. Using automated code checker may detect some faults. 
 
## Testing dimensions
Testing can be characterized in terms of dimensions. These dimensions help to categorized the test types.  

1. What : This dimension describes what are the objectives of the tests. Test objectives vary from one approach to another.
Usually the objectives are the verification or  the validation of functionnal (e.g., portfolio performance) and non-functionnal (e.g., performance, security) requirements.

2. How : This defines how the test objective is achieved. For instance, tests can be either static or dynamic, in isolation or in integration, or knowing the implementation.

3. When : Test can be executed at different moment of the development process. For instance, component testing can be done very early in the development process, while user acceptance test can only be performed when the software is ready for prime time.

4. Who : Different kind of tests are run by different people (e.g., developer, testers , end-users, ...) For instance, component testing can be done by programmers, while user acceptance test are performed by end-users.

### Testing level
Testing levels have been addressed in a number of publications, blog posts and talks [3], [4], [5]. Testing levels describe test types by their quantity and when they occur in the software lifecycle. At the base, tests are done early in the development and extensively. The higher the level, the later the test occurs in the lifecycle. Moreover, while lower levels are usually done the the software supplier, higher levels tend to be performed by the customer.
 
{% include image.html url="/figures/TESTING_LEVELS.png" description="Testing Levels" %}
<br/>

#### Static testing
This sort of testing do not require to execute the code. Tools crawl the code and look for patterns that can lead to fault. Example of tools are [Findbugs](http://findbugs.sourceforge.net/), [PMD](http://sourceforge.net/p/pmd/bugs/). This kind of testing is especially useful to detect complex mistakes involving thread-safety or typing.

#### Unit testing
Test objects are isolated components (classes, packages, programs, ...)  To promote isolation, test objects such as stubs, fakes or mocks can be used. for more information see [Fakes, Stubs, Dummies, Mocks and all that](). These tests happen during development and discovered bugs are fixed right away. Therefore, the management overhead is minimal. Both verification of functional and non.functional requirements can be addressed.

#### Integration testing
Integration testing (a.k.a. assembly testing) verifies the integration between several components. At this level, some components can still be faked to ease deployment and isolation.
Both verification of functional and non.functional requirements can be addressed.

#### API testing
This is the first test level that addresses validation instead of verification. It tests the software using its contracts (API). This is pure blackbox testing usually by using webservices. Tools such as SoapUI are very good at testing the software API and semantics.

#### GUI testing
This level acts on the graphical user interface. Example of tools are [Selenium](http://www.seleniumhq.org/) 

#### System testing
This test level aims the system as a whole with every internal and external components.
Both verification of functional and non.functional requirements can be addressed.

#### Acceptance testing
Both verification of functional and non.functional requirements can be addressed.




### Bibliography
[1 ]Dijkstra (1969) J.N. Buxton and B. Randell, eds, Software Engineering Techniques, April 1970, p. 16. Report on a conference sponsored by the NATO Science Committee, Rome, Italy, 27–31 October 1969. Possibly the earliest documented use of the famous quote.

[2] Antti Valmari. The state explosion problem. In Wolfgang Reisig and Grzegorz Rozenberg, editors, Lectures on Petri Nets I: Basic
Models, volume 1491 of Lecture Notes in Computer Science, pages 429D528. Springer, 1998.

[3] Martin Fowler. TestPyramid. http://martinfowler.com/bliki/TestPyramid.html

[4] Alister Scott. Yet another software testing pyramid. http://watirmelon.com/2011/06/10/yet-another-software-testing-pyramid/
[5] Alister Scott. Introducing the software testing ice-cream cone (anti-pattern). http://watirmelon.com/2012/01/31/introducing-the-software-testing-ice-cream-cone/
