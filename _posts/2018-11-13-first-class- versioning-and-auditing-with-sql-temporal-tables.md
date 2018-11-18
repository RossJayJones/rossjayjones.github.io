---
layout: post
title:  "Implementing Versioning & Audit Trails with SQL Server Temporal Tables and .Net Core"
date:   2018-11-13 09:00:00 +0200
categories: dotnetcore
image:
  feature: jorgen-haland-781448-unsplash.jpg
---
TLDR; [source code available on github][14]{:target="_blank"}

Implementing versioning and audit trails within an application are one of those requirements which crop up over and over again, particularly in enterprise development. At its core a versioning and audit trail solutions require the retrieval of the state of an application data for a specific point in time.

This is a hard problem to solve and is difficult at a number of levels. The first is that the technical solution required for implementing point in time snapshots is complex and requires a lot of experience to implement well. Secondly it is difficult because it often obfuscates our nice clean business model which we are trying to build.

Typically I see the wheels falls off when the solution is implemented at the wrong level of abstraction. If we are too fine grained the solution pollutes the business model and becomes a mess i.e. The infamous `INotifyPropertyChanged` interface. If we are too course we miss some of the richness required or the solution becomes inefficient.

Because it is difficult, and often seen as a non functional requirement, we defer the implementation of these requirements until it is too late. The result is that we land up with a very technical solution which lacks the rich behaviour that the business requires. In the worst case we use software logging tools as an audit trail... you should be ashamed of yourselves :D

If you ask a product owner or domain expert about it, they will likely tell you that it is important. When asked what specifically should be audited/versioned the answer is well... _everything_.

As frustrating as that answer is, sometimes it is the correct one. For the remainder of this post I will introduce [SQL Temporal Tables][13]{:target="_blank"} which is fantastic technical solution for solving the problem at the storage level. More importantly I will explain how to make these concepts first class concerns within your application using [CQRS][10]{:target="_blank"} and Domain [Driven Design][11]{:target="_blank"} which will provide the rich functionality that the business expects.

Before we begin let's take a closer look at audit trails and versioning to see how they are different from each other.

## Audit Trails

According to a quick [google][1]{:target="_blank"} search

> An audit trail (also called audit log) is a security-relevant chronological record, set of records, and/or destination and source of records that provide documentary evidence of the sequence of activities that have affected at any time a specific operation, procedure, or event.

For our purposes this typically means that when something changes we want to record who made the change, what was changed and when was it changed.

This information is most useful when things have gone wrong and an _audit_ needs to be performed to determine where the problem lies. In business terms this means we may need to investigate a customer dispute, an incident of fraud or some other kind of security breach. At a technical level a good audit trail can go a long way to assist in identifying software bugs.

Since audits are most useful when things are going wrong it typically means that they are not looked at until the problem has already occured. When this happens it is helpful to have as much information as possible so that the issue can be effectively diagnosed. The best case scenario is if you have audited _everything_.

A good audit trail could also be used to try and identify problems automatically.

## Versioning

[Versioning][2]{:target="_blank"} on the other hand is subtly different. Versioning in the context of data would be to put a stake in the ground at a specific point in time. You can then retrieve data at a specific point using a version number. 

A version would typically be a first class concern of your business domain and not all domain models require this. In the insurance space good versioning is critical to the day to day processes within the business.

## Git

As a developer you are likely familiar with Git source control. Git is at its core a database which records every change you make to your data as a commit. You can then get a copy of your data at any point in time (or commit). This provides a very good audit trail solution since for any change to the data you can see who did it, what was changed and when it was done. Git quite literally audits _everything_.

You can think of a Git tag or release as a version. You can drop a marker at a specific point in time with a meaningful name. In the software development domain this would typically be a value such as `1.0.0`. This value can be used to communicate which version of the software your customers are using and help you control the roll out of updates to your software.

## The Solution Space

Now that we have a better understanding of what audit trails and versioning are, let's dive into the technical solution. We need to approach solving this problem from two aspects:

* We need a solid technical solution to provide point in time snapshot capabilities
* We need versioning and audit trails to be a first class citizen of our domain model which does not get in our way

In the search for a solid technical solution to the point in time snapshot problem there are two approaches which stand out.

1. Snapshot ([Memento][3]{:target="_blank"}) - An obvious solution would be to take a snapshot of the application state before a change is applied. If implemented naively the snapshots become large and become impractical to work with. An obvious optimisation would be to only create a snapshot for the portion of the application state which has changed. This is how [git][6]{:target="_blank"} works under the hood. Rolling a custom solution to achieve this can be challenging and should not be implemented within your business logic but rather be left to infrastructure to solve.

2. [Event Sourcing][4]{:target="_blank"} - Event Sourcing is an approach to storing state as a series of events. State is restored by replaying events in the order they occurred to get to the latest state. In order to restore data to specific point in time you replay the events up to that point in time. Event sourcing requires quite a big shift in the way we build our applications since events need to be at the core of how your business domain. While this is a very powerful and relevant solution for some domains the facts are that it introduces complexity which feels unnecessary too most of us. Typically solutions like this will work best with purpose built infrastructure components such as [EventStore][5]{:target="_blank"}.

For this post we have chosen to solve the problem using a more traditional approach using snapshots implemented with [SQL temporal tables][7]{:target="_blank"} under the hood. This is a fantastic SQL feature since all the technical complexity is nicely packed away for us and we can focus on the business problem.

## CQRS and Domain Driven Design (DDD)

Before we dive into the implementation details I want to share some thoughts on Command Query Responsibility Segregation ([CQRS][10]{:target="_blank"}).

While the term is a mouthful the concept is simple. Our system will be split into two separate software models. One for the write side (commands) and another for the read side (queries).

This in practical terms means that we have a set of classes that we use for writes, and we have another set of classes which we use for reads.

> I find that when talking about CQRS there is always some confusion around what the write model is. It does not necessarily mean that the write model performs no reads at all. We still need to read data out the database, load it into an object model so we can work effectively with it, then persist those changes back to the DB. It just means that the write model is optimised for write operations, while the query model is optimised for query operations. Your query model should never mutate state on the other hand.

There are many ways to implement a CQRS based system. Some systems go to the extreme where the read and write models are stored in different data stores. While this makes sense in some use cases for the majority of us this creates a ton of unnecessary overhead and is best steered away from.

For the sample we have implemented CQRS by having the write model work against the SQL tables using [Entity Framework Core][8]{:target="_blank"} and the read model based on SQL views using [Dapper][8]{:target="_blank"}.

In our versioning and auditing context the heavy lifting will be done by the queries since they must be aware of the SQL temporal table features. For our write model we rely entirely on the storage engine to do the work and it frees us from needing to know anything about temporal tables.

The second concept I need to share some thoughts on is Domain Driven Design ([DDD][11]{:target="_blank"}). DDD is essentially a way of thinking about the design of your system. It provides strategic and tactical patterns which you can use to build a software domain model. A full breakdown is outside the scope of this post however we need to discuss the concept of an Aggregate which will help us solve the versioning and auditing problem.

> A DDD aggregate is a cluster of domain objects that can be treated as a single unit. An example may be an order and its line-items, these will be separate objects, but it's useful to treat the order (together with its line items) as a single aggregate. - [Fowler][11]{:target="_blank"}

This is important in our context because we will be versioning and auditing the aggregate i.e. if any object within the aggregate changes an audit will be created for the aggregate. If we create a new version a new version of the aggregate as a while as opposed to the individual entities within the aggregate.

## The Sample Business Domain

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

We are going to build an AspNetCore REST API which will support the above. I am only going to highlight the keys aspects of the solution in this post and have omitted some of the mundane plumbing. Refer to the supporting [sample source code][14]{:target="_blank"} for a full view of the sample application.

## The Write Model

We have setup the write model using Entity Framework Core. This is well documented [here][12]{:target="_blank"} and I won't go into details on this.

What I do want to highlight is how the Aggregate controls its invariants via behaviour methods e.g. `AddAddress`, `UpdateAdderss` etc. This is critical in being able to reliably create a contextual audit record.

What is happening here is that when a behaviour method is called on the aggregate it creates an `Audit` with a meaning full message e.g. "Address added" or "Address updated". Since multiple changes could occur in a single transaction or unit of work a single audit can contain multiple messages. This is analogous to a GIT commit message.

The audit is persisted with a `Timestamp`. We can use the timestamp to query the temporal tables and retrieve a snapshot of the aggregate at that point in time.

The crucial but subtle point here is that we have made the audit a first class concern of the domain model. This allows us to reliably recall a list of changes for a given aggregate and view those changes in a human readable way. If we need to view the state of the aggregate at that point in time we can retrieve that _version_ of the aggregate using an audit id.

The exact same mechanism is used for versions. The subtle difference of course is that creating a new version requires an explicit call to `IncrementVersion` by user code. This is analogous to creating a GIT tag or release. 

As a result we have an Audit and a Version as a first class concept  with in our domain model represented by the `Audit` and `Version` classes. In code our domain model appears as follows:

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
    internal Version(string message) : this()
    {
        Message = message;
    }

    public string Message { get; set; }

    public DateTime Timestamp { get; private set; }
}
{% endhighlight %}

## Introduction to SQL Temporal Tables

Up until this point we have done nothing out of the ordinary and have only used vanilla entity framework core. The magic comes in with SQL temporal tables.

The tables behind the domain objects described above have been set up in a specific way to make use of the SQL temporal table feature. The guidance on setting this up is well documented [here][13]{:target="_blank"} and I won't duplicate those details in this post.

The key point is that you will need to setup a history table for every table which you want point in time snapshots for. When a value within a row has been changed the old row is moved into the history table and the update is applied to the row in the main table.

Our aggregate is made up of a number of entities but when a change is made SQL will only _snapshot_ the data which has changed at the row level. This gives us an acceptable level of granularity for creating snapshots of the changes. You will need to be careful about what data you store in a given column. If you are storing huge JSON blobs in a column your history table will become pretty big.

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

Now that we have the write model setup along with the tables to store our aggregate we will take a look at the read model. We will create an abstraction between our write model and read model using a SQL view.

This is critical for keeping our SQL queries clean and simple. When querying a view with the `FOR` keyword SQL will apply that to all the tables referenced inside of the view. The alternative is that we would need to specify the `FOR` keyword on each table referenced in the query and would be error prone and ugly.

The view would appear as follows. Note how there is no reference to the SQL temporal tables feature yet. That will come when we write our query against the view.

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

{% highlight SQL %}
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

## The REST API

Our address book application exposes a REST API for our users to consume. Since the notable portions of the API are on the query side I will only show those controllers here. On the write side the controllers do as you would expect, they accept some data via a POST or PUT and update the data in the database via the domain model.

Our user can retrieve a customer and view their associated addresses for an HTTP get to the following route. We provide an optional `auditId` parameter which will allow the users to retrieve the state of the customer for a specific audit.

{% highlight C# %}
[ApiController]
[Route("api")]
public class GetCustomerController : ControllerBase
{
    private readonly GetCustomerByIdQuery _query;

    public GetCustomerController(GetCustomerByIdQuery query)
    {
        _query = query;
    }

    [Route("customers/{customerId}")]
    [HttpGet]
    public async Task<IActionResult> Execute(int customerId, int? auditId)
    {
        var dto = await _query.Execute(customerId, auditId);
        var results = CreateCustomerResult(dto);
        return Ok(results);
    }

    private static GetCustomerResult CreateCustomerResult(CustomerDto dto)
    {
        var result = new GetCustomerResult
        {
            Name = dto.Name,
            Addresses = dto.Addresses.Select(CreateAddressResult).ToList(),
            Audits = dto.Audits.Select(CreateAuditResult).ToList()
        };
        return result;
    }

    private static GetCustomerAddressResult CreateAddressResult(AddressDto dto)
    {
        var result = new GetCustomerAddressResult
        {
            Id = dto.Id,
            City = dto.City,
            Code = dto.Code,
            Line = dto.Line,
            Province = dto.Province,
            Suburb = dto.Suburb
        };
        return result;
    }

    private static GetCustomerAuditResult CreateAuditResult(CustomerAuditDto dto)
    {
        var result = new GetCustomerAuditResult
        {
            Id = dto.Id,
            Messages = dto.Messages.ToList(),
            Timestamp = dto.Timestamp
        };
        return result;
    }
}
{% endhighlight %}

For versioning we provide a route which will return a list of available versions of the customer and separate route to retrieve the state of the customer for a specified version.

{% highlight C# %}
[ApiController]
[Route("api")]
public class GetCustomerVersionsController : ControllerBase
{
    private readonly GetVersionsQuery _query;

    public GetCustomerVersionsController(GetVersionsQuery query)
    {
        _query = query;
    }

    [Route("customers/{customerId}/versions")]
    [HttpGet]
    public async Task<IActionResult> Execute(int customerId)
    {
        var dtos = await _query.Execute(customerId);
        var results = dtos.Select(CreateGetCustomerVersionsResult);
        return Ok(results);
    }

    private GetCustomerVersionsResult CreateGetCustomerVersionsResult(CustomerVersionDto dto)
    {
        var result = new GetCustomerVersionsResult
        {
            Id = dto.Id,
            Timestamp = dto.Timestamp,
            Message = dto.Message
        };
        return result;
    }
}
{% endhighlight %}

{% highlight C# %}
[ApiController]
[Route("api")]
public class GetCustomerVersionController : ControllerBase
{
    private readonly GetCustomerByVersionIdQuery _query;

    public GetCustomerVersionController(GetCustomerByVersionIdQuery query)
    {
        _query = query;
    }

    [Route("customers/{customerId}/versions/{versionId}")]
    [HttpGet]
    public async Task<IActionResult> Execute(int customerId, int versionId)
    {
        var dto = await _query.Execute(customerId, versionId);
        var results = CreateCustomerResult(dto);
        return Ok(results);
    }

    public static GetCustomerVersionResult CreateCustomerResult(CustomerDto dto)
    {
        var result = new GetCustomerVersionResult
        {
            Name = dto.Name,
            Addresses = dto.Addresses.Select(CreateAddressResult).ToList(),
        };
        return result;
    }

    public static GetCustomerVersionAddressResult CreateAddressResult(AddressDto dto)
    {
        var result = new GetCustomerVersionAddressResult
        {
            Id = dto.Id,
            City = dto.City,
            Code = dto.Code,
            Line = dto.Line,
            Province = dto.Province,
            Suburb = dto.Suburb
        };
        return result;
    }
}
{% endhighlight %}

## Wrapping it up

I believe that SQL temporal tables is a great option for solving this specific problem. It stays out your way when you don't need to think about it and provides powerful capabilities for dealing with point in time snapshots when you do. 

You can probably get 90% of the way there by switching temporal tables on after the fact however hopefully I have shown that by thinking about these concepts up front when designing your domain model provides a lot of rich capabilities which your product owners and domain experts will love you for.

If you want to dig a bit deeper into the solution checkout the The full code sample for this post [here][14]{:target="_blank"}.

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