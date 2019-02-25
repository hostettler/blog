---
layout: post
title: "About Sessions, Conversations, and the Garbage Collector"
date: 2012-06-10 18:51
comments: true
categories: [Java EE 6, JEE6, Java, Session, Conversation, Garbage Collector]
keywords: [Java EE 6, JEE6, Java, Session, Conversation, Garbage Collector]
published: true 
---


A couple of days ago, I had a discussion with a developer about the notion of web conversation.
More precisely, about its utility. During this discussion, we ran into some basic misconceptions
that, IMHO, no web developer must do. Conversation (or flows) are good features of the recent
frameworks (Spring, JSF) because they allow to save information across several user requests without
putting them into the session. For instance, this can be very useful for wizards. Putting these
information into the session asks for manual maintenance, and in particular for manual collection.
This is usual error prone and should be avoided. The problem being that the user rarely logs out
before closing the browser. Thus, ``HttpSession.invalidate()`` does not get a chance to be called
and the session remains active until the timeout occurs (usually 20 minutes later).

The person I was talking with, though that it is not necessary to care about that. In his view, 
the garbage collector will take care of this and that it can be forced. We discuss the matter a bit
deeper and here are some myths that I think must be rebutted.

#### First assumption: I can force the garbage collection

This is not true, actually according to the Java documentation: 
```
Calling the gc method suggests that the Java Virtual Machine expend effort toward recycling unused objects in order to make the memory they currently occupy available for quick reuse. When control returns from the method call, the Java Virtual Machine has made a best effort to reclaim space from all discarded objects.
```

``System.gc()`` **suggests** the garbage collection, it does not enforce it. This signals that you
would like the garbage collector to do its job, but there is no guarantee whatsoever. It is only
possible to enforce garbage collection by writing your own garbage collector, but you cannot expect
all garbage collectors of all the possible VM to act similarly. Therefore, relying on this
assumption is not portable.

#### Second assumption: If I close the browser the session and its objects are collected
Even If it would be possible to enforce the garbage collection, it is of no use as long as there
remains a single reference to the objects to collect. Again, this is absolutely not the case.
Remember that closing the browser does not mean anything to the server. It would, in theory possible
to imagine something with an ajax call that traps the close event of the browser and does a 
``HttpSession.invalidate()`` . This is quite complex to do in a cross-browser manner and gives no
guarantee. Therefore, the data attached to the session will be kept in memory until the session
timeout. This is precisely the beauty of conversation scopes, the developer just tells when the
conversation starts and when it ends. Usually, user do not stop in the middle of a conversation,
they tend to finish it. At least, more often that they click on logout.

#### Third assumption: Anyway this does not represent a huge amount of memory.
Let us take a simple example: a standard business application that, at some point in the business
process, requires a tree for shop selection. This kind of interactions requires several client-server
communications and therefore several requests. A temptation would be to put the tree in the session
scope. Let us say that 10,000 (logical) users log into the application, for instance from 09:00 am to 09:30
am. If each session requires 100 KB (trees are huge even with lazy loading), we end up with 1G memory(only for the tree).
As most of the users do not click on logout, you rely on the timeout to free these objects.

## Conclusion
Using request scope of conversation scope over session scope is a good practice as it frees you
from managing the garbage collection. The code is therefore cleaner and more efficient.
This can be done with JSF's ``@ConversationScoped`` or with Spring Webflow's ``Conversation``.