---
layout: post
title:  "Versioning & Auditing with SQL Server Temporal Tables and .Net Core"
date:   2018-11-13 09:00:00 +0200
categories: dotnetcore
image:
  feature: jorgen-haland-781448-unsplash.jpg
---
TLDR; [source code available on github][14]{:target="_blank"}

Versioning and Auditing are one of those requirements which crop up over and over again, particulaly in enterprise development. If you ask a product owner or domain expert about it, they will likely tell you that it is important. When asked what specifically should be audited/versioned the answer is well... _everything_.

As frustrating as that answer is, sometimes it is the correct one. Lets take a closer look into what auditing and versioning is in order to draw out a more convincing argument.

## Audit Trails

According to a quick [google][1]{:target="_blank"} search

> An audit trail (also called audit log) is a security-relevant chronological record, set of records, and/or destination and source of records that provide documentary evidence of the sequence of activities that have affected at any time a specific operation, procedure, or event.

For our purposes this typically means that when something changes we want to recored who made the change, what was changed and when was it changed.

This information is most useful when things have not gone to plan and an _audit_ needs to be performed to determine what went wrong. In business terms this means we may need to investigate a customer dispute, an incident of fraud or some other kind of security breach.

Since audits are most useful when things are going wrong it typically means that they are not looked at until the problem is identified. When this happens it is helpful to have as much information as possible so that the issue can be effectively diagnosed. The best case scenario is if you have audited _everything_.

## Versioning

[Versioning][2]{:target="_blank"} on the other hand is subtly different. Versioning in the context of data would be to put a stake in the ground at a specific point in time. You can then retrieve data at a specific point using a version number.

This im my opinion is a first class concern of your business domain as opposed to auditing which can be seen as a cross cutting function. It is a snapshot of your data at a specific point in time.

Typically the technical solution of creating a point in time snapshot can be used to build both a versioning and/or auditing feature.

## Git

As a developer you are likely familiar with Git source control. Git is at its core a database which records every change you make to your data as a commit. You can then get a copy of your data at any point in time (or commit). This provides a very good auditing solution since for any change to the data you can see who did it, what was changed and when it was done. Git quite literally audits _everything_.

You can think of a Git tag or release as a version. You can drop a marker at a specific point in time with a meaningful name. In the software development domain this would typically be a value such as `1.0.0`. This value can be used to communicate which version of the software your customers are using and help you control the roll out of updates to your software.

What we want to achieve in the remainder of this post is to build a solution that will provide this functionality on top of our business data stored within a SQL database.

## Technical Solutions

We need to approach solving this problem from 2 apsects namely we need a technical solution to the problem, then we need to figure out how to introduce the concepts into our business domain.

In order to solve the technical aspect we need to figure out all the moving parts which will make auditing and versioning possible at the storage layer. We have a number of options available to us and we will briefly take a look at some of them.

1. Snapshot ([Memento][3]{:target="_blank"}) - An obvious solution would be to take a snapshot of the state before a change is applied. If implemented naively the snapshots become big and waste a lot of space. A better approach is to store only the parts of your data which have changed. This is how [git][6]{:target="_blank"} works under the hood. Roling a custom solution to achieve this can be challenging.

2. [Event Sourcing][4]{:target="_blank"} - Event Sourcing is an approach to storing state as a series of events. State is restored by replaying events in the order they occured to get to the latest state. In order to restore data to specific point in time you replay the events up to that point in time. Snapshots are also used in this approach as an optimisation mechanism. You can take a snapshot after a certain number of events have occured to allow you to start replaying from a snapshot and not have to replay all events. Typically this implementation can become very complicated and is best handled by dedicated storage systems such as [EventStore][5]{:target="_blank"}.

We chose to solve the problem using the snapshot approad discussed above using [SQL temporal tables][7]{:target="_blank"} under the hood.

This is a good solution since all the techinical details around storing a snapshot of only the rows which have changed are taken care of by the storage engine.

## CQRS and Domain Driven Design (DDD)

Before we dive into the implementation details I want to share some thoughts on Command Query Responsibility Segregation ([CQRS][10]{:target="_blank"}).

While the term is a mouthful the concept is sinple. Our system will be split into 2 separate software models. One for the write side (commands) and another for the read side (queries).

This in practical terms means that we have a set of classes that we use for writes, and we have another set of classes which we use for reads.

There are many ways to implement a CQRS based system. Some systems go to the extreme where the read and write models are stored in different data stores. While this makes sense in some use cases I believe for the majority of us this creates a ton of unncessary overhead and is best steered away from.

For the sample we have implemented CQRS by having the write model work against the SQL tables using [Entity Framework Core][8]{:target="_blank"} and the read model based on SQL views using [Dapper][8]{:target="_blank"}.

In our versioning and auditing context the heavy lifting will be done by the queries since they must be aware of the SQL temporal table features. For our write model we rely entirely on the storage engine to do the work and it frees us from needing to know anything about temporal tables.

The second concept I need to share some thoughts on is Domain Driven Design ([DDD][11]{:target="_blank"}). DDD is essentially a way of thinking about the design of your system. It provides strategic and tactical patterns which you can use to build a software domain model. A full explanation is outside the scope of this post however we need to discuss the concept of an Aggregate which will help us solve the versioning and auditing problem.

> A DDD aggregate is a cluster of domain objects that can be treated as a single unit. An example may be an order and its line-items, these will be separate objects, but it's useful to treat the order (together with its line items) as a single aggregate. - [Fowler][11]{:target="_blank"}

This is important in our context because we will be versioning and auditing the aggregate i.e. if any object within the aggregate changes an audit will be created for the aggregate. If we create a new version the version will be created for the aggregate.

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

If any change is made to a given customer that change should be audited and there should be a mechanism to view the state of the data for a specific audit entry.

Separately we should be able to create a named version of a customer which can be retrieved using the version number.

A change to customer means any of the following:

* A property on the customer is changed
* An address is added or removed
* A property on an address is changed

In the instance we have modeled the Customer as an Aggregate where the addresses are entities which belong to the aggregate.

We are going to build an AspNetCore REST API which will support the above.

## The Write Model

We setup the write model using Entity Framework Core. This is well documented [here][12]{:target="_blank"} and I wont go into details on this.

What I do want to highlight is how the Aggregate controls its invariants via behaviour methods e.g. `AddAddress`. This is critical in being able to reliably create a contextual audit record.

What is happening here is that when a behaviour method is called on the aggregate it creates an `Audit` with a meaning full message e.g. "Address added" or "Address udpated". Since multiple changes could occur in a single transaction or unit of work a single audit can contain multiple messages. This is analogous to a GIT commit message.

The audit is persisted with a `Timestamp`. We can use the time stamp to query the temporal tables and retrieve a snapshot of the aggregate at that point in time.

The crucial but subtle point here is that we have made the audit a first class concern of the domain model. This allows us to reliably recall a list of changes for a given aggregate and view those changes in a human readable way. If we need to view the state of the aggregate at that point in time we can retrieve that _version_ of the aggregate using an audit id.

The exact same mechanism is used for versions. The subtle difference of course is that creating a new version requires an explicit call to `IncrementVersion` by user code. This is analogous to creating a GIT tag or release.

{% highlight C# %}
public class Customer
{
    private Audit _currentAudit;

    private readonly List<Address> _addresses;
    private readonly List<Audit> _audits;
    private readonly List<Version> _versions;

    internal Customer()
    {
        _addresses = new List<Address>();
        _audits = new List<Audit>();
        _versions = new List<Version>();
    }

    public Customer(string name) : this()
    {
        Name = name;
        OnChanged("Customer created");
    }

    public int Id { get; private set; }

    public string Name { get; private set; }

    public IReadOnlyCollection<Address> Addresses => _addresses;

    public IReadOnlyCollection<Audit> Audits => _audits;

    public IReadOnlyCollection<Version> Versions => _versions;
    
    public void Update(string name)
    {
        Name = name;
        OnChanged("Name changed");
    }

    public Address AddAddress(
        string line,
        string suburb,
        string city,
        string province,
        string code)
    {
        var address = new Address(
            customer: this,
            line: line,
            suburb: suburb,
            city: city,
            province: province,
            code: code);
        _addresses.Add(address);
        OnChanged("Address added");
        return address;
    }

    public void UpdateAddress(int addressId,
        string line,
        string suburb,
        string city,
        string province,
        string code)
    {
        var address = _addresses.Single(a => a.Id == addressId);
        address.Update(
            line: line,
            suburb: suburb,
            city: city,
            province: province,
            code: code);
        OnChanged($"Address {addressId} updated");
    }

    public void IncrementVersion(string message)
    {
        var version = new Version(
            message: message);
        _versions.Add(version);
    }

    private void OnChanged(string message)
    {
        if (_currentAudit == null)
        {
            _currentAudit = new Audit();
            _audits.Add(_currentAudit);
        }

        _currentAudit.AddMessage(message);
    }
}
{% endhighlight %}

{% highlight C# %}
public class Address
{
    internal Address()
    {
        
    }

    public Address(Customer customer,
        string line,
        string suburb,
        string city,
        string province,
        string code) : this()
    {
        Customer = customer;
        Line = line;
        Suburb = suburb;
        City = city;
        Province = province;
        Code = code;
    }

    public int Id { get; private set; }

    public Customer Customer { get; private set; }

    public string Line { get; private set; }

    public string Suburb { get; private set; }

    public string City { get; private set; }

    public string Province { get; private set; }

    public string Code { get; private set; }

    internal void Update(
        string line,
        string suburb,
        string city,
        string province,
        string code)
    {
        Line = line;
        Suburb = suburb;
        City = city;
        Province = province;
        Code = code;
    }
}
{% endhighlight %}

{% highlight C# %}
public class Audit
{
    private List<string> _messages;

    internal Audit()
    {
        _messages = new List<string>();
    }

    public DateTime Timestamp { get; private set; }

    public IReadOnlyCollection<string> Messages => _messages;

    internal void AddMessage(string message)
    {
        _messages.Add(message);
    }
}
{% endhighlight %}

{% highlight C# %}
public class Version
{
    internal Version()
    {
        
    }

    internal Version(string message) : this()
    {
        Message = message;
    }

    public string Message { get; set; }

    public DateTime Timestamp { get; private set; }
}
{% endhighlight %}

## SQL Temporal Tables

Up until this point we have done nothing out of the ordinary. The magic comes in with SQL temporal tables.

The tables behind the domain objects described above have been set up in a specific way to make use of the SQL temporal table feature. The guidance on setting this up is well documented [here][13]{:target="_blank"} and I wont duplicate those details here.

The key point is that you will need to setup a history table for every table which you want versioned, and you turn history tables on for the main table.

For example the address history table is created with the SQL command below.

{% highlight SQL %}
CREATE TABLE [dbo].[AddressHistory]
(
  [Id] INT NOT NULL, 
  [Line] NVARCHAR(255) NOT NULL, 
  [Suburb] NVARCHAR(255) NOT NULL, 
  [City] NVARCHAR(255) NOT NULL, 
  [Province] NVARCHAR(255) NOT NULL, 
  [Code] NVARCHAR(255) NOT NULL, 
  [CustomerId] INT NOT NULL,
  [SysStartTime] DATETIME2 NOT NULL,
  [SysEndTime] DATETIME2 NOT NULL
)
GO
CREATE CLUSTERED COLUMNSTORE INDEX IX_AddressHistory ON [dbo].[AddressHistory];
GO
CREATE NONCLUSTERED INDEX IX_AddressHistory_ID_PERIOD_COLUMNS ON [dbo].[AddressHistory] ([SysEndTime], [SysStartTime], [Id]);
GO
{% endhighlight %}

And the main address table is created with the following SQL command.

{% highlight SQL %}
CREATE TABLE [dbo].[Address]
(
  [Id] INT NOT NULL PRIMARY KEY IDENTITY, 
  [Line] NVARCHAR(255) NOT NULL, 
  [Suburb] NVARCHAR(255) NOT NULL, 
  [City] NVARCHAR(255) NOT NULL, 
  [Province] NVARCHAR(255) NOT NULL, 
  [Code] NVARCHAR(255) NOT NULL, 
  [CustomerId] INT NOT NULL,
  [SysStartTime] DATETIME2 GENERATED ALWAYS AS ROW START NOT NULL,
  [SysEndTime] DATETIME2 GENERATED ALWAYS AS ROW END NOT NULL,
  CONSTRAINT [FK_Address_Customer] FOREIGN KEY ([CustomerId]) REFERENCES [Customer]([Id]),
  PERIOD FOR SYSTEM_TIME ([SysStartTime], [SysEndTime])
)
WITH (SYSTEM_VERSIONING = ON (HISTORY_TABLE = [dbo].[AddressHistory]))
{% endhighlight %}

## The Read Model

The read model is built around a SQL view. This is important since when we create the data from a specific point in time we do not want to have to specify the timestamp for each table in the query.

Instead if we create a view we can specify the time stamp once when querying the view and the storage engine will ensure that the time stamp is applied to all versioned tables with view.

The view would appear as follows. Note how there is no reference to the SQL temporal tables feature here.

{% highlight SQL %}
CREATE VIEW [dbo].[v_Customer] AS 
SELECT
    [Id],
    [Name],
    JSON_QUERY((
      SELECT
      [Address].[Id],
      [Address].[Line],
      [Address].[Suburb],
      [Address].[City],
      [Address].[Province],
      [Address].[Code]
      FROM
          [Address]
      WHERE
          [Address].[CustomerId] = [Customer].[Id]
      FOR JSON PATH
    )) AS [Addresses]
FROM
    [Customer]
{% endhighlight %}

The SQL query to retrieve the customer for a specific audit id would appear as follows. Notice how `SYSTEM_TIME AS OF @Timestamp` only needs to defined in one place and it will be applied to both the `Customer` and `Address` tables within the view.

{% highlight C# %}
DECLARE @Timestamp DATETIME;
SELECT 
    @Timestamp = [Timestamp]
FROM 
  [Audit]
WHERE
  [Id] = @AuditId;

SELECT
    [Id],
    [Name],
    [Addresses]
FROM
    [v_Customer]
FOR
    SYSTEM_TIME AS OF @Timestamp
WHERE
    [Id] = @CustomerId;
{% endhighlight %}

## Wrapping it up

You can see that we have not had to do a lot of work to get a fully featured versioning and auditing system in place with SQL temporal tables. It largely stays out the way and allows us to build applications how we are used to doing.

The full code sample for this post can be found [here][14]{:target="_blank"}. The repository holds a working example of address book application using AspNetCore.

[1]:https://en.wikipedia.org/wiki/Audit_trail
[2]:https://en.wikipedia.org/wiki/Versioning
[3]:https://en.wikipedia.org/wiki/Memento_pattern
[4]:https://martinfowler.com/eaaDev/EventSourcing.html
[5]:https://eventstore.org/
[6]:https://git-scm.com/book/en/v2/Getting-Started-Git-Basics
[7]:https://docs.microsoft.com/en-us/sql/relational-databases/tables/temporal-tables?view=sql-server-2017]
[8]:https://docs.microsoft.com/en-us/ef/core/
[9]:https://github.com/StackExchange/Dapper
[10]:https://martinfowler.com/bliki/CQRS.html
[11]:https://martinfowler.com/bliki/DDD_Aggregate.html
[12]:https://docs.microsoft.com/en-us/ef/core/
[13]:https://docs.microsoft.com/en-us/sql/relational-databases/tables/creating-a-system-versioned-temporal-table?view=sql-server-2017
[14]:https://github.com/RossJayJones/versioning-and-auditing-with-temporal-tables