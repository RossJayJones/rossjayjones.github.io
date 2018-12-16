---
layout: post
title:  "Implementing Versioning & Audit Trails with SQL Server Temporal Tables and .Net Core"
date:   2018-11-13 09:00:00 +0200
categories: dotnetcore
image:
  feature: jorgen-haland-781448-unsplash.jpg
---

Inevitably when building enterprise software you will be required to implement audit trail and/or versioning functionality. At its core, versioning and audit trails rely on point-in-time snapshots. My team and I recently investigated different ways to implement this. We opted for a solution built with ASP.NET Core and SQL Server Temporal Tables. Here’s why, and how we implemented it.

## What we needed to build

I work as a developer for a South African insurance company. Since protecting sensitive personal data and preventing fraud are such key concerns to my company, our core insurance system needed reliable auditing. When data is changed, we needed to be able to see:

- Who made the change
- What was changed
- When it was changed and
- Why it was changed

> An audit trail is a record of change to your data which serves as proof of compliance with the applications business rules. It can be used to assist with identifying problems in your software and spotting fraudulent transactions among other things.

In addition, we needed a mechanism to version the data so that we could support complex business processes which rely on the data at specific points in time.

> Versioning involves assigning a version number to your data at a specific point in time. This allows data to be changed without breaking business process which relies on earlier versions of the data.

For this article, let's look at a simplified example of what my team needed to build. Our sample system will keep a registry of customers along with a list of addresses for properties they need to insure.

```
+----------+         +----------+
|          |         |          |
| Customer + ----->> + Address  |
|          |         |          |
+----------+         +----------+
```

For security purposes, any change to a customer or an associated address must be audited. Additionally, for any given change, the full state of the customer at that point in time must be retrievable via a [REST API][15]{:target="_blank"} call to be used by a front end or another services. A change to a customer would mean any of the following:

- A property on the customer is changed
- An address is added or removed
- A property on an address is changed

Separately, to support business processes which rely on data at a specific point in time, we should be able to create a version of a customer that can be retrieved via a REST API using the version number.

## Why it was difficult to do this

The technical solution required for implementing point-in-time snapshots is complex and requires a lot of time and experience to implement well.

*If your implementation is too fine-grained:* You will land up with a complex technical solution where your code is polluted with details associated with tracking changes on individual properties, like the infamous `INotifyPropertyChanged` interface.

*If your implementation is too coarse:* You risk sacrificing system efficiency. For example, you may be tempted to just serialize your entire object graph and store that as a JSON dump for every change which is made.

Because it is difficult, and is often seen as a non-functional requirement, we defer the implementation of these requirements until it is too late. The result is that we land up with a very technical solution which lacks the rich behaviour that the business requires. In the worst case, we use the output of development logging tools as an audit trail... You should be ashamed of yourselves! :D

## Our options — and why we picked SQL Server Temporal Tables

There are two solutions to the point-in-time snapshots problem which require a mention.

1. Snapshot ([Memento][3]{:target="_blank"}) - An obvious solution would be to take a snapshot of the application state before a change is applied. If implemented naively, the snapshots become large and impractical to work with. An obvious optimisation of this would be to create a snapshot for only the portion of the application state which has changed. This is how [git][6]{:target="_blank"} works under the hood. Rolling a custom solution to achieve this can be challenging and we decided it should be left to the storage infrastructure to solve. This ensures that our domain logic remains pure and free from unnecessary complexity.

2. [Event Sourcing][4]{:target="_blank"} - Event Sourcing is an approach to storing state as a series of events. The state is then restored by replaying events in the order that they occurred to materialize the latest state. To restore data to a specific point-in-time, you replay the events up to the time that you need. Event sourcing requires a big shift in the way applications are built since events are at the core of the business data model. While this is a very powerful and relevant solution for some domains, the fact is that it introduces complexity which felt unnecessary for our requirements. Typically, solutions like this will work best with purpose-built infrastructure components such as [EventStore][5]{:target="_blank"}.

We will implement a snapshot solution using [SQL temporal tables][7]{:target="_blank"}. It is a great point in time snapshot implementation which we can leverage with relatively little effort, but it does require some upfront design. It uses snapshots at the row level so, when a row is updated, a snapshot of the row is copied over to the associated history table giving us snapshot granularity at the row level. 

This was the best option for us because the technical challenges of doing this are hidden away and exposed via a few additional SQL statements which makes it very simple to work with.

## How we implemented SQL Server Temporal Tables

While it would be possible to turn on the temporal tables and call it a day, there are some concepts which must be understood for auditing and versioning to be first-class citizens of our business domain. This is necessary to:

- Unlock the rich functionality required within this domain, and 
- Achieve an appropriate separation of concerns within the implementation.

We chose to make use of Domain Driven Design and implement the CQRS pattern to achieve this.

#### Command Query Responsibility Segregation: 

In the sample system, we will be using the Command Query Responsibility Segregation ([CQRS][10]{:target="_blank"}) pattern. While the term is a mouthful, the concept is simple. It means that our system will be split into two separate software models:

The write side for commands: We will build a model backed by SQL tables using [Entity Framework Core][8]{:target="_blank"}. There will be no need to utilise Temporal Table features directly in this layer and will be using plain old Entity Framework.

The read side for queries: This is another set of classes which will be used for reads. This model will be built over SQL views using [Dapper][8]{:target="_blank"}. This is the layer which will be doing the heavy lifting with the help of Temporal Tables.

#### Domain Driven Design

Domain Driven Design ([DDD][11]{:target="_blank"}) is essentially a way of thinking about the design of your system. It provides strategic and tactical patterns which you can use to build a software domain model. A full explanation is outside the scope of this post, however we need to understand the concept of an aggregate which will help us solve the versioning and auditing problem.

> A DDD aggregate is a cluster of domain objects that can be treated as a single unit. An example may be an order and its line-items. These will be separate objects, but it's useful to treat the order (together with its line items) as a single aggregate. - [Fowler][11]{:target="_blank"}

In our sample system we have one aggregate which is the customer. The DDD concept of an aggregate is important in this context because we will be versioning and auditing it. So, if any object within the aggregate changes or a new version is created, a snapshot will be created for the whole aggregate.

### Step 1: The write model

I have setup the write model using Entity Framework Core. This is well documented [here][12]{:target="_blank"} and I won't go into details on this.

Because the customer is an aggregate it can control its invariants via behaviour methods e.g. `AddAddress`, `UpdateAddress` etc. This is what gives us the hook to create a contextual audit record.

When a behaviour method is called on the aggregate, it creates an `Audit` with a meaningful message e.g. "Address added" or "Address updated". Since multiple changes could occur in a single transaction, or unit of work, a single audit can contain multiple messages. This is analogous to a GIT commit.

The audit is persisted with a `Timestamp` which we will use to query the temporal tables and retrieve a snapshot of the aggregate at that point in time.

Because we have used DDD to model our domain objects we have an `Audit` object which  is used to reliably recall a list of changes for a given aggregate and view those changes in a human readable way. If we need to view the state of the aggregate at that point in time we can retrieve that _snapshot_ of the aggregate using an audit id.

The exact same mechanism is used for versions. The subtle difference of course is that creating a new version requires an explicit call to `IncrementVersion` by user code. This is analogous to creating a GIT tag or release.

As a result, we have an Audit and a Version as a first-class concept within our domain model, represented by the `Audit` and `Version` classes, which are not bogged down with any point-in-time snapshot technical complexity.

In code our domain model appears as follows:

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

### Step 2: Configuring SQL Temporal Tables

Up to this point, we have done nothing out of the ordinary and have only used vanilla entity framework core. The magic comes in with the SQL temporal tables.

The tables behind the domain objects described above have been set up in a specific way to make use of the SQL temporal table feature. The guidance on setting this up is well documented [here][13]{:target="_blank"}.

The gist is that you will need to create an associated history table for each table that you want to track for changes. When a value within a row changes, the old row is moved into the history table and the update is applied to the row in the main table.

Our aggregate is made up of a number of entities but, when a change is made, SQL Server will only take a _snapshot_ of the rows which have changed preventing wasted disk space. When joining together a number of tables, SQL Server will take care of selecting the correct version of the data from each table for the specified point in time. 

> You will need to be careful about what data you store in a given column. If you are storing huge JSON blobs in a column then you will land up wasting disk space since that entire column will be moved into the history table.

For example, the address history table is created with the following SQL command:

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

And the main address table is created as follows:

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

### Step 3: The read model

Now that we have the write model setup along with the tables to store our aggregate, we will take a look at the read model. We will create an abstraction between our write model and read model using a SQL view will help keep our SQL queries clean and simple.

When querying a view with the `FOR` keyword, SQL will apply the temporal query to all the tables which are referenced inside of the view. If do not use a view we would need to specify the `FOR` keyword on each table referenced in the query. This would be error prone and ugly.

The SQL command to create the view would appear as follows:

> Note how there is no reference to the SQL temporal tables feature yet, that will come when we write our query against the view.

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

The SQL query to retrieve the customer for a specific audit would appear as follows: 

> Notice how `SYSTEM_TIME AS OF @Timestamp` only needs to defined in one place and will be applied to both the `Customer` and `Address` tables within the view.

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

### Step 4: The REST API

Our sample system will expose it’s functionality using a REST API which can be consumed by a front end or some other service. We will concentrate on the query endpoints here since that is where the interesting stuff happens. The controllers responsible for writes do as you would expect; they accept some data via a [POST][15]{:target="_blank"} or [PUT][15]{:target="_blank"} and update the data in the database via the domain model.

Our users can retrieve a customer and view their associated addresses via an [HTTP GET][15]{:target="_blank"} to the following route: 

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

For versioning, the provided route will return a list of available customer versions and a separate route for retrieving the state of the customer for a specified version.

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

## Wrapping up

I believe that SQL Server Temporal Tables is a great option for solving this specific problem. It is transparent to the parts of your application which are not directly concerned with the technical aspects of point-in-time snapshots, and provides simple and powerful point-in-time snapshot capabilities for the parts of your application which are.

You can probably get 90% of the way there by enabling temporal tables as an afterthought. But by thinking about these concepts when designing your domain model, there is potential for rich capabilities that will make your product owners and domain experts love you.

If you want to dig a bit deeper into the solution, check out the full code sample for this post [here][14]{:target="_blank"}.

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
[15]:https://en.wikipedia.org/wiki/Representational_state_transfer