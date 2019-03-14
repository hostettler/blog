---
layout: post
title: "Microservice Architecture - Part 3 (Diving into microservices)"
date: 2019-03-04T22:51:20+01:00
published: false
---


## Introduction
In this chapter, we will dive into the concept of microservices. First, we will discuss why the community came with yet another architecture paradigm. Secondly, we will 
look at some definitions and the main properties of such an architecture. Then, we will detail some of the technologies microservice architecture leverage to deliver maximum value. 
After that, we will analyse the pros and cons of this architecture.
Finally, we will discuss some of the architecture patterns related to microservices.


## Why (yet) another architecture paradigm
The microservice architecture can be seen as a reaction to the monolith architecture. In particular, the fact that the bigger the application, the slower the pace of change. 
Modifying a monolith requires to
rebuilt it completly and to ship it in one block. Furthermore, by its very nature, a monolith tend to favor API leaks and to decrease modularity. 
Finally, having one block means that scalability happens at
that granularity which might lead to a waste of resources as all the modules of the monolith will be scaled at the same time whereas not all the modules might 
need the same type of scalability or no scalability at all,

## Definition(s)
At this core, microservice is an architecture (MSA) that is an evolution of the service oriented architecture (SOA). It inhnerits from SOA several key concepts. The most important one being
that business added value is delivered by combining a collection loosely coupled services.

Reusing the definition of services from SOA, a service has the [following properties](https://en.wikipedia.org/wiki/Service-oriented_architecture):
- It logically represents a business activity with a specified outcome.
- It is self-contained.
- It is a black box for its consumers (and the communication between consumer and provider is formalized by a contract).
- It may consist of other underlying services.

These properties apply to microservices as well. The main differentce between SOA and MSA is in the granularity of the services and some opinionated implementation choices 



## Technologies
Microservices also leverage new technologies to foster its core benefits.
Containers
#### Relation to cloud native technologies

## Pros and Cons

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


## Bibliography
- [1] N. Poulton, (2017) Docker Deep Dive
- [2] Turnbull, J. (2014). The Docker Book: Containerization is the new virtualization. 
- [3] Mauro Vocale, Luigi Fugaro (2018). Hands-On Cloud-Native Microservices with Jakarta EE
- [4] Raghuram Bharathan, (2015). Apache Maven Cookbook