---
layout: post
title:  "Versioning & Auditing with SQL Server Temporal Tables and .Net Core"
date:   2018-11-13 09:00:00 +0200
categories: dotnetcore aspnetcore mvc dotnet sql
image:
  feature: jorgen-haland-781448-unsplash.jpg
---
TLDR; [source code available on github](https://github.com/RossJayJones/versioning-and-auditing-with-temporal-tables){:target="_blank"}

Versioning and Auditing are one of those requirements which crop up over and over again, particulaly in enterprise development. If you ask a product owner or domain expert about it, they will likely tell you that it is important. When asked what specifically should be audited/versioned the answer is well... _everything_.

As frustrating as that answer is, sometimes it is the correct one. Lets take a closer look into what auditing and versioning is in order to draw out a more convincing argument.

## Audit Trails

According to a quick [google][1] search

> An audit trail (also called audit log) is a security-relevant chronological record, set of records, and/or destination and source of records that provide documentary evidence of the sequence of activities that have affected at any time a specific operation, procedure, or event.

For our purposes this typically means that when something changes we want to recored who made the change, what was changed and when was it changed.

This information is most useful when things have not gone to plan and an _audit_ needs to be performed to determine what went wrong. In business terms this means we may need to investigate a customer dispute, an incident of fraud or some other kind of security breach.

Since audits are most useful when things are going wrong it typically means that they are not looked at until the problem is identified. When this happens it is helpful to have as much information as possible so that the issue can be effectively diagnosed. The best case scenario is if you have audited _everything_.

## Versioning

[Versioning][2] on the other hand is subtly different. Versioning in the context of data would be to put a stake in the ground at a specific point in time. You can then retrieve data at a specific point using a version number.

This im my opinion is a first class concern of your business domain as opposed to auditing which can be seen as a cross cutting function. It is a snapshot of your data at a specific point in time.

Typically the technical solution of creating a point in time snapshot can be used to build both a versioning and/or auditing feature.

## Git

As a developer you are likely familiar with Git source control. Git is at its core a database which records every change you make to your data as a commit. You can then get a copy of your data at any point in time (or commit). This provides a very good auditing solution since for any change to the data you can see who did it, what was changed and when it was done. Git quite literally audits _everything_.

You can think of a Git tag or release as a version. You can drop a marker at a specific point in time with a meaningful name. In the software development domain this would typically be a value such as `1.0.0`. This value can be used to communicate which version of the software your customers are using and help you control the roll out of updates to your software.

What we want to achieve in the remainder of this post is to build a solution that will provide this functionality on top of our business data stored within a SQL database.

## The Business Domain

For the sake of this post lets define a simple business domain which we will build a system for. This system will have full auditing and versioning functionality as described above.

Our system will keep a registry of customers with a list of addresses per customer... an address book if you will :D

```
+----------+         +----------+
|          |         |          |
| Customer + ----->> + Address  |
|          |         |          |
+----------+         +----------+
```

If any change is made to a given customer that change should be audited and there should be a mechanism to view the state of the data at a specific audit.

Separately we should be able to create a named version of a customer which can be retrieved using the version number.

A change to customer means any of the following:

* A property on the customer is changed
* An address is added or removed
* A property on an address in changed

We are going to build an AspNetCore REST API which will support the above.

## Domain Driven Design & CQRS

Before we do that I just want to take a moment to explain some concepts which will allow us to build this solution effectively.

This problem can be solved by purely implementing the technical solution. In this case that would mean turning on SQL temporal tables and be done with it. While this will work and will get you 90% of the way, putting some additional thought into modeling the system carefully will provide auditing and versioning as a first class concept within your application as opposed to a technical after thought.

[1]:https://en.wikipedia.org/wiki/Audit_trail
[2]:https://en.wikipedia.org/wiki/Versioning