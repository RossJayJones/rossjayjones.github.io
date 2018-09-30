---
layout: post
title:  "Resource Scheduling Apps with RavenDB 4.0"
date:   2018-03-03 13:00:00 +0200
categories: document_databases architecture dotnet
image:
  feature: time.jpg
---

I was recently tasked with building resource scheduling functionality into a system which uses a document database as the back end store.

For the sake of this blog post lets say we were required to book a meeting room for a meeting. In order to do this we would need to find a time slot where the meeting room is available and reserve it for the duration of the meeting. In a big organisation meetings rooms become something of a contentious resource, so the system would need assume that there could be multiple people trying to book the same room with overlapping time slots at any given time. 

The system we were working on uses [RavenDB](https://ravendb.net/) as the back end store. The RavenDB indexing system, which is used to query documents, is based on an eventually consistent paradigm. This means that if I were to query the index at any point in time the results i receive _may_ be stale. 

Normally this is acceptable since since by the time the web page loads the data you seeing is stale anyway. In this case however I needed to be able to book the meeting room with ACID guarantees to ensure that a time slot for a meeting room would not be able to be double booked.

RavenDB provide ways to allow you to prevent stale reads from an index, however in a busy system this can be an expensive operation. This ruled out simply querying the index for bookings within a time frame to test whether the meeting room is available.

Loading documents by ID in RavenDB however is guaranteed to be consistent. That together with the optimistic concurrency enforced at the document level allows us to ensure that I cannot double book a meeting room.

## Time is infinite, RAM is not

Since time is infinite the only way to model it in a software system is by considering it in chunks. When thinking about how to model this with a document database system we have the following options:

- Store all reservations for a given resource in a single document - This might make sense if the number of reservations for a resource remains small. Once you go past a couple of hundred reservations it will become expensive to load and modify this document. In our case this was not going to work well for very long.

- Store each reservation in it's own document and rely on indexes to query availability of the calendar. As discussed above, this does not work for our requirements due to the eventual consistency issues.

- The third approach, and the one we went for, is to break the calendar up into chunks and have a document which represents a chunk of time. In our case we decided to have a document represent a day of the calendar. So there would be one document per resource per day.

![Chunks]({{site.url}}/blog/assets/calendar-chunks.png "Chunks")

## Managing the Chunks

There are a couple of things which need to be considered with this approach.

- We need to be able to deal with reservations which span multiple days.

- When working with the calendar it needs to be aware of its bounds. For instance if I load up the calendar for 1 Jan, then try to book an appointment for 2 Jan, the calendar needs to either load up more data so that it can double check the requested time slot is available, or throw an error indicating that the operation is not allowed and additional data would need to be loaded up.

This got me thinking about how to go about providing an intuitive interface to the calendar service without having to worry about all the mechanics of managing the chunks of time. Consider the following C# class

{% highlight C# %}

public class Calendar {

	public Calendar(DateTime begin, DateTime end) {
		...
	}
	
	public bool IsAvailable(DateTime begin, DateTime end) {
		...
	}

	public void Reserve(string reservationId, DateTime begin, DateTime end) {
		...
	}

	public void Cancel(string reservationId) {
		...
	}
}

{% endhighlight %}

What I wanted from the Calendar is to allow the user to work with a chunk of time without having to understand the `chunking` mechanism behind the scenes. The calendar class needs to protect the system by enforcing the time boundaries that it knows about in order to ensure double bookings cannot occur. The constructor accepts the boundaries as arguments which are then used to perform the validation when trying to create a reservation.

The calendar class then has knowledge of all the reservations during the given time period so that it can before the necessary validation.

So the trick is then to create a mechanism which is able to construct a calendar instance given a date range. In the sample I created a `CalendarApplicationService` who's job it is to create a Calendar from a number of chunks of time, and break a Calendar back into chunks when updating it.

{% highlight C# %}
public class CalendarApplicationService {

	public Calendar Get(string id, DateTime begin, DateTime end) {
		...
	}

	public void Update(Calendar calendar) {
		...
	}
}
{% endhighlight %}

### Storing the Chunks

If we choose our document id's carefully this task is trivial with a document database or key/value store.

We chose an ID scheme for each day of the calendar as follows

```
{calendarId}_{yyyy}_{mm}_{dd}
```

So given a calendar ID and a date range I can construct a list of ID's then load all the chunks up with one network call. We can then combine all the chunks together to provide a single view of the calendar. The sample contains a helper class which performs these functions

{% highlight C# %}
private class CalendarChunkHelper
{
	private readonly Reservation _reservation;

	private CalendarChunkHelper(Reservation reservation)
	{
		_reservation = reservation;
	}

	public static IEnumerable<CalendarChunk> SplitIntoChunks(Calendar calendar)
	{
		return calendar.Reservations.Select(entry => new CalendarChunkHelper(entry))
			.SelectMany(helper => helper.SplitIntoChunks())
			.GroupBy(x => x.Id)
			.Select(entry => new CalendarChunk(entry.Key, entry.ToList()));
	}

	public static IEnumerable<string> CreateChunkIds(string id, DateTime begin, DateTime end)
	{
		var ids = new List<string>();
		while (begin <= end)
		{
			var chunkId = $"{id}-{begin.Year}-{begin.Month}-{begin.Day}";
			begin = begin.AddDays(1);
			ids.Add(chunkId);
		}
		return ids;
	}

	private IEnumerable<CalendarChunkEntry> SplitIntoChunks()
	{
		var slices = new List<CalendarChunkEntry>();
		var begin = _reservation.Begin.Date;
		var end = _reservation.End.Date;

		while (begin <= end)
		{
			var slice = new CalendarChunkEntry
			{
				Id = $"{_reservation.CalendarId}-{begin.Year}-{begin.Month}-{begin.Day}",
				Begin = Max(begin, _reservation.Begin),
				End = Min(begin.AddHours(23).AddMinutes(59).AddSeconds(59), _reservation.End),
				CalendarEntryId = _reservation.Id
			};
			slices.Add(slice);
			begin = begin.AddDays(1);
		}

		return slices;
	}

	private DateTime Max(DateTime a, DateTime b)
	{
		if (b > a)
		{
				return b;
		}

		return a;
	}

	private DateTime Min(DateTime a, DateTime b)
	{
		if (b > a)
		{
				return a;
		}

		return b;
	}
}

{% endhighlight %}

Where we do not find a document with the given id it indicates that there were no entries for that day. i.e. a document only needs to be created for days which have reservations booked.

The documents which ultimately get stored may appear as follows

{% highlight C# %}

public class Reservation
{
	public Reservation(string id,
		string calendarId,
		DateTime begin,
		DateTime end)
	{
		Id = id;
		CalendarId = calendarId;
		Begin = begin;
		End = end;
	}

	public string Id { get; set; }
	public DateTime Begin { get; set; }
	public DateTime End { get; set; }
	public string CalendarId { get; set; }
}

public class CalendarChunk
{
	public CalendarChunk()
	{
			
	}

	public CalendarChunk(string id, 
		List<CalendarChunkEntry> entries)
	{
		Id = id;
		Entries = entries;
	}

	public string Id { get; set; }

	public List<CalendarChunkEntry> Entries { get; set; }

	public void Update(IEnumerable<CalendarChunkEntry> entries)
	{
		Entries = entries.ToList();
	}
}

public class CalendarChunkEntry
{
	public string Id { get; set; }
	public DateTime Begin { get; set; }
	public DateTime End { get; set; }
	public string CalendarEntryId { get; set; }
}

{% endhighlight %}

> It is important to understand that there may be a lot more information pertaining to the reservation. My suggestion is that this information is stored in a separate document which represents the reservation, while the calendar simply contains a reference to the reservation id i.e. it's the calendars responsibility to control reservations for a resource for a given time, while its the reservations job to store information about the reservation.

When a reservation spans multiple days you will land a CalendarChunkEntry in multiple CalendarChunks.

## Wrapping Up

This approach has been working well for us so far. Document databases provide a lot of flexibility and speed but may require a bit more thought around how your data is stored. I hope this post has given some insight on how dealing with time based systems could be handled with document databases.

A full working sample is available on [Git Hub](https://github.com/RossJayJones/resource-scheduling-with-document-databases) which will hopefully fill in any gaps which this post may have left.
