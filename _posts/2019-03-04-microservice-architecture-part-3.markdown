---
layout: post
comments: true
title: "Microservice Architecture - Part 3 (Diving into microservices)"
date: 2019-03-04T22:51:20+01:00
published: false
---


## Introduction
In this chapter, we will dive into the concept of microservices. First, we will discuss why the community came with yet another architecture paradigm. Secondly, we will 
look at some definitions and the main properties of such an architecture. Then, we will detail some of the technologies microservice architecture leverage to deliver maximum value. 
After that, we will analyse the pros and cons of this architecture. Finally, we will discuss some of the architecture patterns related to microservices.


## Why (yet) another architecture paradigm
The microservice architecture can be seen as a reaction to the monolith architecture. In particular, the fact that the bigger the application, the slower the pace of change. 
Modifying a monolith requires to
rebuilt it completly and to ship it in one block. Furthermore, by its very nature, a monolith tend to favor API leaks and to decrease modularity. 
Finally, having one block means that scalability happens at
that granularity which might lead to a waste of resources as all the modules of the monolith will be scaled at the same time whereas not all the modules might 
need the same type of scalability or no scalability at all.

Breaking the monolith apart in small pieces, enables to make architectural decision at that level. For instance, to choose the most appropriate programming language or technology
for each task. That being said, adding  heteregoneity comes at a price in terms of necerrary infrastructure and thus complexity.

## Definition(s)
At this core, microservice is an architecture (MSA) that is an evolution of the service oriented architecture (SOA). It inhnerits from SOA several key concepts. The most important one being
that business added value is delivered by combining a collection loosely coupled services.

Reusing the definition of services from SOA, a service has the [following properties](https://en.wikipedia.org/wiki/Service-oriented_architecture):
- It logically represents a business activity with a specified outcome.
- It is self-contained.
- It is a black box for its consumers (and the communication between consumer and provider is formalized by a contract).
- It may consist of other underlying services.

These properties apply to microservices as well. The main differentce between SOA and MSA is in the granularity of the services and some opinionated implementation choices. 
Microservices have not been invented in isolation, they leverage the other advances and trends in Softwares Engineering such as Agile, DevOps concepts, and Cloud.
This led to add the following properties:
- each service must be elastic, resilent, composible, minimum, and complete.
- each services must take automation, deployment, and testing
- each is specialized in one thing and in doing that thing right



## Technologies
Microservices also leverage new technologies to foster its core benefits.
Containers
#### Relation to cloud native technologies


## Some core properties
#### Single Database per service
#### Domain per service
#### Service Granularity

## architecture patterns

#### Command Query Responsibility Segregation (CQRS)
#### Message Bus
#### API Composition / API Gateway

## Cross-Cutting Concerns
#### Security
#### Logging / Tracability

## Pros and Cons

#### Distributed system
#### Deployment complexity
#### Increased resource consumption
As a microservice architecture entails many more instances (e.g., VMs, JVMs) to run that its monolitic counterpart. Furthermore, entities are very often replicated between the instances (to increase loose coupling).
All of that lead to higher overall resources (memory and CPU) consumption. This is composenated by the fact that more resources are available than years ago before the cloud era.


## Bibliography
- [1] N. Poulton, (2017) Docker Deep Dive
- [2] Turnbull, J. (2014). The Docker Book: Containerization is the new virtualization. 
- [3] Mauro Vocale, Luigi Fugaro (2018). Hands-On Cloud-Native Microservices with Jakarta EE
- [4] Raghuram Bharathan, (2015). Apache Maven Cookbook