---
layout: post
title: "Verification & Validation"
date: 2012-05-18 10:51
comments: true
categories: [test, verification, validation, definition]
keywords: [test, verification, validation, definition]
published: true 
---

I've read many articles (tutorials, books, courses, forums...) about
Verification & Validation. One think that really strikes me is the lack of consistency and the
fuzziness among the definitions or more precisely among the interpretations. As I do think that
something cannot be understood completely if it is not be expressed clearly, I propose a mini-serie of blog posts to clarify Verification & Validation and the associated
activities. I will try to give simple and consistent (at least in my point of view) definitions. Sure, this is a rehash of well-known stuff but putting everything at the same place seems worth to me. I do not pretend to know best and thus feel free to comment on this post to correct me or to suggest other explanations and definitions. My only purpose is to help to better understand these words and the underlying concepts.

## Verification & Validation
Most of the literature on the subject distinguish between **Verification** and **Validation**. The
usual way (taken from Barry Boehm) to sum up the difference between the two is to say that:

``Verification  consist in checking that we are building the product right, and validation
 consists in checking building the right product.``

I am not very satisfied with this definition. Although it sounds good, it raises the question of
what it does mean to built a product right and what the right product is. Let's go back to the roots
by looking at the definitions of these words in the [Merriam-Webster](http://www.merriam-webster.com/) dictionary.
<!---
###Oxford dictionary
**Verification**: the process of establishing the truth, accuracy, or validity of something.  
**Validation**: check or prove the validity or accuracy of something.

###Cambridge dictionary
**Verification**: to prove that something exists or is true, or to make certain that something is correct.  
**Validation**: to make something officially acceptable or approved, especially after examining it.
-->

**Verification**: the act or process of "establishing the truth, accuracy, or reality of a claim".  
**Validation**: the act or process of "making legally valid".

<!--- All three dictionaries agree (as far as I understand it) on the definition of verification.
The 
[Oxford] (http://oxforddictionaries.com/) dictionary does not really distinguish the two words.

[Cambridge] (http://dictionary.cambridge.org/) and [Merriam-Webster]
(http://www.merriam-webster.com/), on the other hand, describe **validation** as the act of making
something legal or official. As French is my mother tongue and both **verification** and **validation**
come from the middle French, I checked the definitions in a French dictionary ([Larousse](http://www.larousse.com/en/dictionaries/french/)).
The Larousse's definition also makes a clear distinction that is along the same lines as the Cambridge 
and the Merriam-Webster dictionary.  
-->
To summarize, my understanding is that the general sense of **verification** is to prove a claim
whereas **validation** aims at declaring something legal or official (i.e. something that comply to a
set of rules or regulations). Of course this raises the question of what, in the context of software
systems, is a claim and what is legal/official. This leads us to the next section that defines terms
such as requirements, and specification. 

## Requirements & Specification
<!---
###Oxford dictionary
**Requirement**:   
**Specification**: 

###Cambridge dictionary
**Requirement**:   
**Specification**: 
-->
Again, let's lookup the meaning of these words in the [Merriam-Webster](http://www.merriam-webster.com/) dictionary:

**Requirement**: the need for a particular purpose; depend on for success or survival.  
**Specification**: the act of describing or identifying something precisely or of stating a precise requirement.

A requirement expresses a need upon which the success of the project depends. Whereas, a
specification is a precise description of a requirement. Now let us look at the standard definitions
in Software Engineering.

## Software Requirements & Specification
As a software systems is usually developed in response to someone needs (or supposed needs), it
must satisfy that person, that group of person, or that organization needs. Furthermore, it must
often comply to a set of regulations or other expectations. This leads us to the following concept:

**Requirements**  represent what the stakeholders expect from the system. It is important to note
   that the stakeholders can be end-users, power-users or even internal or external regulatory
   entities such as government agencies that enforce laws and regulations. Stakeholder requirements
   do not take the feasibility into account nor do they take technical details into account. They
   only state their problem or expectations.

As it is not possible to have access to all stakeholders at any time during the development process
(for instance, to ask questions), it is necessary to capture their requirements in a persistent form.
In the following, I call the role that capture the stakeholder requirements: the business analyst.
 
**Specifications**  capture in a written form what a business analyst understood from the
 requirements. This written form is usually expressed in natural language or is a mixed of natural
 language and semi-formal, or formal description. The specifications describe implementable requirements and how to meet them. Therefore, they propose a solution to the problem.

Interestingly enough, there is the same kind of semantical difference between **requirements** and **specifications**
as between **verification** and **validation**. This is the basis that helps to understand the subtle
yet important difference between the two terms, in particular in the context of the development of
software systems.

 It is important to understand is that the specifications are derived from the interpreted **requirements**
 and therefore **specifications** cannot be sound and complete. In other words, there may be things
 that are over-specified (not sound) and things that are under-specified (not complete).

Let's imagine for a second that a perfect business analyst did capture exactly what the
stakeholders had in mind. This perfect specification must still be implemented and therefore we must
ensure that implementation does satisfy the specification (and by transitivity the stakeholders
needs).

A software system does reflect the stakeholder requirements if the following hypotheses hold:

1.  All stakeholders expresses their needs in a sound and complete set of requirements;
2.  a business analyst captures these requirements in a sound and complete specification (with respect to the requirements);
3.  the development team understands and implements this specification in a sound and complete way;

Of course non of these hypotheses hold in a real industrial setting but we have to admit them to be
able to realize a system. In order to minimize the risk of diverging from the stakeholders
requirements to much we need a process to ensure that the final system agrees to a certain degree to
the stakeholders requirements. This a the role of verification and validation.


## Software Verification & Validation
The figure below illustrates the difference between **verification** and **validation** and where
they take place in the development process. Building a software system consists in capturing the
stakeholders' requirements into a persistent description, called **informal specifications** (e.g.,
Use Cases, storyboards, ...). This specifies **what** problems the system addresses. At this stage a
number of interpretation mistakes may have occurred leading to incorrect specifications. To limit
these problems, the **informal specifications** must be explained to the user and validated.
Developers require non-ambiguous specification to work on. Thus, a design (e.g., UML diagrams) are
derived from the **informal specifications** . Similarly, the quality assurance team derives a **formal
specification** (e.g, Petri nets, test cases, metrics, ...). The design and later the implementation
are then verified with respect to this **formal specification**. To a certain extend, the formal specification
must itself be validated by the stakeholders.


{% include image.html url="/figures/V&V.png" description=" Verification and Validation " %}


**Verification**    stands for the process of evaluating whether a software system satisfies the **specified
requirements**. On the design level, it can be achieved by simulation or model checking. On the implementation
   level, it can be achieved by different kind of testing, code review, static analysis, and
   metrics. In other words: "Did we build the system right (with respect to specified
   requirements)?".

**Validation**    stands for the process of evaluating whether a software system satisfies the **stakeholder requirements**
   and **intended uses** . This is, for instance, achieved by **User Acceptance Tests** and by
   submitting the final system to certification authorities. In other words: "Did we build the right
   (with respect to stakeholder requirements) system?".

<!---
Validation is necessary because there will always be a gap (or discrepancy) between what the customer wants, and what has been captured and expressed in the requirements. There is inevitably some loss, or corruption, of information in its transmission between the customer and the analyst/developer.
The essential point is that the customer or user must be directly involved in validation.
--->

## Conclusion
Stakeholders cannot be available at any time of the development. Therefore, it is necessary to
build a persistent view (i.e., specification) of their needs (i.e., requirements). The activity of
checking whether the design and the implementation fulfill the specification is called 
**Verification**
Some details may be lost in the translation from the requirements to the specification. Therefore,
at each phase of the development of a software systems, we must check that what has been specified
and later implemented during that phase reflect the stakeholders' expectations. This activity is
called **validation** . Integrating these two concepts as first class citizen, is one of the major
benefits of agile methods. Because the interpretation of the **stakeholder requirements** and later
the translation from the **informal specifications** to the **formal specifications** may be
incomplete or incorrect, it is necessary to often go back to the customer or the stakeholders to
confirm that the product is on the good track. This is one of the major benefits of iterative and
incremental development.



<!---
The [Food and Drug Administration]() 

**Software verification**  provides objective evidence that the design outputs of a particular phase
 of the software development life cycle meet all of the specified requirements for that phase.
 Software verification looks for consistency, completeness, and correctness of the software and its
 supporting documentation, as it is being developed, and provides support for a subsequent
 conclusion that software is validated. Software testing is one of many verification activities
 intended to confirm that software development output meets its input requirements. Other
 verification activities include various static and dynamic analyses, code and document inspections,
 walkthroughs, and other techniques.

**Software validation**  is a part of the design validation for a finished device, but is not
 separately defined in the Quality System regulation. For purposes of this guidance, FDA considers
 software validation to be ``confirmation by examination and provision of objective evidence that
 software specifications conform to user needs and intended uses, and that the particular
 requirements implemented through software can be consistently fulfilled.'' In practice, software
 validation activities may occur both during, as well as at the end of the software development life
 cycle to ensure that all requirements have been fulfilled. Since software is usually part of a
 larger hardware system, the validation of software typically includes evidence that all software
 requirements have been implemented correctly and completely and are traceable to system
 requirements. A conclusion that software is validated is highly dependent upon comprehensive
 software testing, inspections, analyses, and other verification tasks performed at each stage of
 the software development life cycle. Testing of device software functionality in a simulated use
 environment, and user site testing are typically included as components of an overall design
 validation program for a software automated device.

IEEE-829-2008 3.1.53 validation: (A) The process of evaluating a system or component during or at
the end of the development process to determine whether it satisfies specified requirements.
(adopted from IEEE Std 610.12-1990 [B3] ) (B) The process of providing evidence that the software and
its associated products satisfy system requirements allocated to software at the end of each life
cycle activity, solve the right problem (e.g., correctly model physical laws, implement business
rules, or use the proper system assumptions), and satisfy intended use and user needs.

3.1.54 verification: (A) The process of evaluating a system or component to determine whether the
products of a given development phase satisfy the conditions imposed at the start of that phase.
(adopted from IEEE Std 610.12-1990 [B3] ) (B) The process of providing objective evidence that the
software and its associated products comply with requirements (e.g., for correctness, completeness,
consistency, and accuracy) for all life cycle activities during each life cycle process
(acquisition, supply, development, operation, and maintenance), satisfy standards, practices, and
conventions during life cycle processes, and successfully complete each life cycle activity and
satisfy all the criteria for initiating succeeding life cycle activities (e.g., building the
software correctly).


1. [IEEE-829-2008]()
2. [ISTQB foundation]() and [advanced]() syllabus
3. [ISO 9000:2000]()
4. [ISO 8402:1994]()
5. [ISO/IEC 14971-1 and IEC 60601-1-4]() 
6. [General Principles of Software Validation; Final Guidance for Industry and FDA Staff]()
-->
