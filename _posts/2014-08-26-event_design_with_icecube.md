---
layout: post
title:  "Event Design"
date: 2014-08-26
categories: design
---
### Simple Event ###

The simplest event has only a title and a schedule. The shedule
describes when the event occurs. eg title:"*My Birthday*",
schedule:"*Yearly, on 17th July, duration 24hrs*".

![](http://yuml.me/diagram/plain/class/%252F%252F%20Cool%20Class%20Diagram,%20%5BSimplestEvent%7Ctitle:String;schedule:IceCube::Schedule%5D;%5Bnote:The%20schedule%20is%20serialised%20automatically%20by%20active%20record.%20%7Bbg:wheat%7D%5D.png)

The Icecube::Schedule is serialised to_yaml by adding a serialized
attribute in the model. The attribute is stored as a string (Text
field) in the database.

{% highlight ruby %}

class Event < ActiveRecord::Base
  serialize :schedule, IceCube::Schedule
end

{% endhighlight %}

###Optimise by time

The optimised version adds the *next_occurence* attribute, which is
also used as a key in an index. Events now have a timeline.

*next_occurence* is not intended to be set explicitly by the
application, it's a read-only convenience variable, created from the
schedule. It can be arranged that the attributed is updated
automatically in the background when the row is saved. You can read
about using Active record *callback* here, [Rails - Active Record
Callbacks](http://guides.rubyonrails.org/active_record_callbacks.html#available-callbacks).

Pages that list events by time now have an index to optimise their
querries.

![](http://yuml.me/diagram/plain/class/%5BOptimsedEvent%7Ctitle:String;schedule:IceCube::Schedule;next_occurence%20:Datetime%7CIndex%20on%20next_occurence%5D.png)

Since events like scrums/hangouts are usually updated just after the
event closes, to update the youtube_url, or the actual_duration or
paticipents, so too will the *next_ccurrence* of the event.

Some events may not update on close. So there is still a need to visit
every event in the table to update the *next_occurence*. But this need
only occure once per day.

Something along the lines of :

    defun daily_event_refresh(date)
      Events.all.each do |e|
        e.save
      end
    end
{{:ruby}}

* future events (by next_occurence) can be ignored 

* past events that have passed their last occurence time can also be
ignored. To make the query efficient maybe, a *is_expired* column is
required on the event table to be able to filter those out at the
Database level.

####Difference

This is a little different to the current model. Currently, events are
generated on the fly. 

So to generate the Event Index Page, (about 50 occurences) the app
uses the following logic.

Foreach event of in table, a schedule is created, the next 10
occurrences are generated and stored in an array. (up to a maximum of
100 occurrences), the array is sorted by time, finally the top-n are
returned.

> So even this is slightly flawed, if there are more than 10 daily
> events, then events 11 and greater, are ignored and not
> displayed. 

The advantage of generating on the fly are :

* it does flesh out a small set (5 or so) into an reasonably sized
event queue for display (~50 occurrences ~ 10 days).

* it doesn't require a DB hit

* however it does scan on Events table to get all events.

Reading the Database and having a time index

Simply selects the next events in time order from an epoch, limited
time or count. This means that if there few events, say 5, then only
the those next 5 occurences would be displayed. There's no
*fill-into-the-future*.

However, reading the table via the new index, begins to work better
when the event/pairing count increases. 

I don't know the website page view statistics, however I would imagine
that the Event Index page is probably amongst the top popular and
these changes simplify this page's data retrieval.

In addition, the countdown timer (top left corner) on the website has
to find the next event time and it currently also scans the event
table (this time only generating one next occurence), which are then
sorted, in order to find the next scrum time for count-down.

I'll discuss filling into the future after the next section.

###Attributes

Events can gain attributes over time, eg. hoa_url, actual_duration,
participants, Aggendas, etc. An attribute actually belongs to an event
occurrence (event instance), ie. each meeting has a distinct video
url.

IceCube doesn't provide an occurrence-hash or any means to uniquely
identify each occurence in the schedule, so I'll fall-back to using
the date-time string of the start_time of the occurrence as an id,
*occurrence_start* and store it in a Datetime column in the
EventInstanceAttribute table. (which I've called the AttributeItem in
the relationship diagram below).

The index, becomes occurrence_start + event_type for the index.

![](http://yuml.me/diagram/plain;dir:LR/class/%5BOptimsedEvent%7Ctitle:String;schedule:IceCube::Schedule;next_occurence%20:Datetime%7CIndex%20on%20next_occurence%5D++-0..*%3E%5BAttributeItem%7Cid:integer;event_type:integer;event_id:integer;occurrence_start:Datetime;type:integer;value:String%7Cindex;occurrence_start;event_type%7Cid (primary);%5D.png)

Note. 

* As soon as an attribute is added to an event, (by insertion into
AttributeItem table) an occurrence exists in the database.

* event_type has been added for convenience to allow types of events to
be selected directly without a join.

###Prefill-forward

We could arrange to populate occurences with a pseudo attribute, just
to cause a presence. A *Presence* attribute, would allow us to create
occurrences into the furture. 

Using the model like this would allow the query behind the Event index
pages into a simple select via the index. Past events can also be
retrieved.

###Schedule Updates - What happens

When there are updates to the schedule, it will affect the keys and
indexes.

I'll look at this in more detail.

####Event

* The schedule is part of the event and will be serialised during
  `event.save`

* *next_occurrence* can be updated via before_save from the schedule.

That's it for events.

####AttributeItems

* Items on past occurences do not need to be updated, they have
  recorded what has happened and don't need to be touched.
  
* Future presence attributes can just be deleted and re-created (if
  required) from the new schedule.

* Future real attributes deserve more thought because the time of the
  event is it's key.

I think the easiest way to deal with these are to update each existing
occurence time with the new time, in sequence. But we may end up with
more or fewer occurences as a result of the schedule change.

####Example
 
We have a 3 desgin meetings every 2 days, each with their own aggenda
file attached.

    Event                         Attribute Items

    "Swimming pool design meeting"
                                 <- 21/8/2014 10:00 utc, File:"Agenda Scope.txt"
                                 <- 23/8/2014 10:00 utc, File:"Agenda Capture.txt"
                                 <- 25/8/2014 10:00 utc, File:"Agenda Prioritise.txt"

If the schedule changed to: 

#### More occurences, ie Six daily meetings.

Then how do we fill them ? 

Suggestion:

    "Swimmingpool design meeting"
             new_next            -> 21/8/2014 10:00 utc, File:"Agenda Scope.txt"
             new_next+1          -> 22/8/2014 10:00 utc, File:"Agenda Capture.txt"
             new_next+2          -> 23/8/2014 10:00 utc, File:"Agenda Prioritise.txt"
                                    [Note following Presence attr are optional ]
             new_next+3          -> 24/8/2014 10:00 utc, Presence:
             new_next+4          -> 25/8/2014 10:00 utc, Presence:
             new_next+5          -> 26/8/2014 10:00 utc, Presence:

#### Fewer occurences, ie Two daily meetings.

Then how do they merge ? 

    "Swimmingpool design meeting"
                                 -> 21/8/2014 10:00 utc, File:"Agenda Scope.txt"
                                 -> 22/8/2014 10:00 utc, File:"Agenda Capture.txt"
                                 -> 22/8/2014 10:00 utc, File:"Agenda Prioritise.txt"

The 22nd August has two aggenda's attached.

In each case only the user really knows what is best.

##Sub classing Events

There might be a requirement to sub-class an event if different
behaviour is needed. A *hoa_event* for example, may have an
extended finishing workflow (profile updates, youtube url, kudos, etc).

STI (Single Table Inheritence) was suggested by Sam, on chat in the
summer, which is built into Rails' Active Record implimentation.

![sdf](http://yuml.me/diagram/plain;dir:LR/class/[OptimisedSTIEvent%7Ctitle:String;schedule:IceCube::Schedule;next_occurrence:Datetime;type:String%7CIndex:%20next_occurence],%20[OptimisedSTIEvent]%5E--[Sprint],%20[OptimisedSTIEvent]%5E--[HoaEvent])

An attribute named type has been added to facilitate STI, read more at [api-rubyonrails.org](http://api.rubyonrails.org/classes/ActiveRecord/Base.html#class-ActiveRecord::Base-label-Single+table+inheritance).

##What can we improve

Well it's interesting, because what we have is similar but not really
the same.

event

: lacks the schedule, but does contian variables to construct a new
one.

hangouts 

: Are similar to AttributeItem, but lacks an index. 

Has a uid column, but lacks a occurence datetime (although I have
discovered that created_at is currently used for this). Now this
doesn't make sense if for example the EventInstance is given an
Aggenda document, a few days before the meeting will take place, then
created_at is no use. (infact I think both created_at and updated_at
are both redundant.

![](http://yuml.me/3fc6bfa9)

##Change summary

###Store the Schedule

We should store the schedule in the event and not chuck it away.

Reason:

* It becomes available when the event is instantiated.

    e = Event.find(:id event_id)
    e.schedule.next(10) # produces the next 10 occurrences # simple :-)

* IceCube::Schedule is has a well documented interface, and I'm sure
  we could not do better ourselves in terms of efficiency and
  performance. (Isn't this why we have chosen the Gem in the first
  place?)

###Add timeline

Add *next_occurence* as described above. As the quantity of events
increase, being able to retrieve by time becomes more important to the
application and event index/filtering pages.

###Simplify our interface

* remove our invented schedule variables and use the stored
  IceCube::Schedule directly. Candidates : `start_time, repeats,
  repeats_every_n_weeks, repeats_weekly_each_days_of_week_mask,
  repeat_ends repeat_ends_on`.

* If we need to extract something from the schedule so that it is
  stored as a column in the database to facilitate filtering, make
  sure this is well understood, so that the next person, doens't alter
  this logic. The schedule is the source.

* remove some of our support methods that really only serve to
  confuse. If a helper method is required, re-check naming to avoid
  confusion.

* remove *start_time* because it is ambiguous what it means
  (next_occurence ?, epoch_of_event_series/singleton ?, creation_date
  ?) This might be what I call *event.next_occurence* or
  *attribute.occurence_start*.

* How is the `uid` used? Could/Should it become a datetime for the
  start_of_instance.

* What does `slug` do for us ? And we check why we have an index on
  this column. How often do we access an event by it's slug ?

* What is the url in the event used for ? Should this become an
  attribute?

* What is the `timezone` used for ? Who's timezone is it? I thought
  all times in here are UTC ?

* `duration`, probably should be renamed to `estimated_duration`, then
  `actual_duration` can be another occurence attribute. I don't think
  this is actually used for anything in the application other than
  description for the event and to give an event a probable end
  time. So maybe it is better to get rid of this and use the duration
  in the schedule.

###Move Attributes

* create the AttributeItem model as described above. (Should it be
  named EventInstanceAttribute instead ?

This gives us somewhere to store occurrence attributes and can have as
many or few as we want.

We don't require a schema change to accomodate new attributes, which
we haven't thought about yet.

Possible Attributes types

* Paticipant
* hoa_url
* youtube_url
* youtube_duration
* Project Planning meeting Indicator
* Client Meeting Indicator
* remind participants via email Indicator

###Remove hangouts

* refactor the hangouts to use AtributeItems.

###Other possible improvements

* a means to eliminate expired events, ie events (both single and
  repeating) where they have reach their last occurrence, because they
  are bounded by time or count.

* a task which runs at regular intervals (daily), to populate *missing
  forward events* in the attribute table.

* archive really old events, to keep the tables lean. (once per month).

###Future - STI

Maybe sub-class with STI ?
