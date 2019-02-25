---
layout: post
title: "A (small) hitchhiker's guide to Data Warehousing"
date: 2016-02-10 14:07:42 +0100
comments: true
categories: 
---

In my current assignment, I had the opportunity to discuss with Data Warehouse (DWH) experts about its integration with the rest of the information system.  I noticed that not every stakeholders (included Data Warehouse professionals) use the same vocabulary.
During the discussions, people raised words such as ["Data Warehouse"](#dwh), ["Data Marts"](#dmt), [ODS](#ods), ["Data Lake"](#lak) and so on. Some of the words were used interchangeably which does not help to follow the discussion. As I was not familiar  with several  of them, I decided to do my homework and to come up with a small glossary to provide a common ground for further discussions.

>Disclaimer : I am not an expert in the field, I only tried to come up with a couple of definitions to get a common ground for further discussions.  Please experts in the field, help me to improve this!

## Business Intelligence (B.I.)

> Business intelligence is a set of methodologies, processes, architectures, and technologies that transform raw data into meaningful and useful information. It allows business users to make informed business decisions with real-time data that can put a company ahead of its competitors.
>
> <cite>Boris Evelson - Forrester</cite>

Business Intelligence Systems usually (but not always) rely on a Data Warehouse to provide information out of raw operational data.
The following diagram shows the relationships between the different levels involved in making a  decision.
Information emerge from consolidated data thus helping the user to improve her knowledge on a given subject. Using this knowledge, she then can make a informed decision.
{% include image.html url="/figures/B.I.png" description="How B.I. helps Decision making" %}

Business Intelligence Systems have the following properties:

* They leverage raw heterogeneous operational data 
* They enable multi-dimensional information and operations on it
* They are driven by the business
* They have to be performant and must not interfere with daily operations

## Data Warehouse (DWH) <a id="dwh"></a>
DWHs are central repositories of integrated data from one or more disparate sources. Its purpose is to organize and homogenize data into information. User can then leverage this information into knowledge and therefore make informed decision (see B.I.).

There are  three main approaches on how to build a data warehouse. 

### William Inmon's approach

According to [William Inmon](http://www.amazon.com/Building-Data-Warehouse-W-Inmon/dp/0471141615) that originally coined the term "Data Warehouse", a data warehouse has the following properties:

- Subject oriented : This implies that data are organized around the business and not around the sources. For instance, several accounting data sources are consolidated into one accounting data warehouse. The purpose of which is to letting information emerge out of data.
- Integrated : Coming from different sources, data must be standardized to enable consistency and thus letting information emerge. For instance, customer identification must be normalized across different sources.
- Non-volatile : Once in the data warehouse, data must not be altered. Therefore, data is available for future comparison. 
- Time-variant : Changes made on data over time are tracked. For instance, each and every change to a customer country of residence are tracked.

Inmon’s model follows a top-down approach. First, a complete (enterprise wide) Data Warehouse (DW) is created in [third normal form](https://en.wikipedia.org/wiki/Third_normal_form) (3NF : avoiding duplication and having referential integrity) and then, if required, [datamarts](#dmt) (DMT) are provisioned out of the DW. Datamarts in Inmon’s model are in 3NF from which the [OLAP](https://en.wikipedia.org/wiki/Online_transaction_processing) cubes are built.
For Inmon, data quality and coherency is paramount and thus the 3NF in the DW and the DMs.

{% include image.html url="/figures/Imnon.png" description="The Imnon Top-Down model" %}

### Ralph Kimball's approach
According to [Kimball](http://www.amazon.com/The-Data-Warehouse-Toolkit-Dimensional-ebook/dp/B004GXBZFQ) another prominent actor in this field, "Data Warehouse is a system that extracts, cleans, conforms, and delivers source data into a dimensional data store and then supports and implements querying and analysis for the purpose of decision making." This definition does not contradict Inmon's properties. The difference lies in the architecture.

Kimball’s model follows a bottom-up approach. First, some (Datamarts)(#dmt) (DM) emerge directly sourced from [OLTP](https://en.wikipedia.org/wiki/Online_transaction_processing) (Online Transaction Processing Systems) systems usually follow the company processes and organization.  
The [Datamarts](#dmt) are either in 3NF ([OLAP cubes](#cube) are built on top of them) or de-normalized star schemas. 

{% include image.html url="/figures/Kimball.png" description="The Kimball Bottum-Up model" %}

### The Hybrid model - A typical architecture

Both of these approaches have their pros and cons. Kimball’s model is easy to start with because of the bottom-up approach and hence you can start small and scale-up eventually. Moreover, the ROI is usually better with Kimball’s model. 
Because of this approach it is difficult to created re-usable data structures and operations (extraction) for different [datamarts](#dmt). Finally, you may end-up with consistency problems. 
On the other hand, Inmon’s approach is structured and easier to maintain at the cost of being rigid and more expensive. 

Real-life DWH implementation often end-up using a hybrid architecure. The following architecture relies on the following biulding blocks:

* a staging layer to extract and sanitize data
* an [ODS](#ods) to enable "close to the operation" data mining
* an 3NF DWH with full history to enable the creation unanticipated [datamarts](#dmt)
* [datamarts](#dmt) that rely either on 3NF or on the star schema for better performances

{% include image.html url="/figures/DWHArchitecture.png" description="A typical Hybrid Architecture" %}


### Data Mart (DMT) <a id="dmt"></a>
A datamart is essentially a basic building block of the data warehouse. 
It is subject-oriented subset of a Data Warehouse. Data mart does not explicitly imply the presence of a multi-dimensional technology such as OLAP and data mart does not explicitly imply the presence of summarized numerical data.


### Operational Data Store (ODS) <a id="ods"></a>
An operational data store (ODS) is building block of Data warehouse used for immediate reporting with operational data.  An ODS contains lightly transformed and lightly integrated operational data with a (rather) short time window. The ODS is usually used when looking for specific events (settling a banking movement or looking for a specific operation). Full history is available in the DWH.

### OLAP Cubes <a id="cube"></a>
OLAP Cubes are multidimensional arrays of data comming from a relational database. It enables operations such as slicing and dicing (projection), drill down/up, roll-up. A datamart relying on a star schema provides equivalent functionalities. However, in a cube every projection/aggregation are pre-computed (this enables to discover new patterns) whether in a star schema only some projections/aggregations (the one you know are interrested) are pre-computed.

# Conclusion
Business Intelligence is a set of practices/methodologies that leverage raw data into decisions. Data warehouses, data marts and cubes are building blocks used to build Busines Intelligence system.
