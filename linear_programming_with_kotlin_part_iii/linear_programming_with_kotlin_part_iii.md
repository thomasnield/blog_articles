# Linear Programming with Kotlin Part III - Generating Multi-day Schedules

In [Part I of this series](http://tomstechnicalblog.blogspot.com/2018/01/kotlinforoperationalplanningandoptimiza.html) I introduced binary programming with [Kotlin](http://kotlinlang.org/) and [ojAlgo](http://ojalgo.org/). In [Part II], I introduced continuous variables and optimization concepts. Sometime after that, I started building [okAlgo](https://github.com/thomasnield/okAlgo/blob/master/README.md) which is a Kotlin idiomatic extension to ojAlgo. But I digress. In this section, I am going to present something more ambitious and useful: generating mutli-day schedules. This can be applied to scheduling problems such as staffing, manufacturing, transporation, event planning, and even classroom allocation.

It is one thing to create an app that allows you to input events into a calendar. It is another for it to automatically generate the calendar of events for you! Rather than relying on iterative brute-force tactics to fit classes into a schedule (which can be dysmally and unacceptably inefficent), we can achieve this magic one-click generation of a schedule using the power of mathematical modeling.

In this article, we will generate a weekly university class schedule against one classroom. We will plot the occupation state grid on two dimensions: classes vs timeline. If we wanted to schedule against multiple rooms, that would be three dimensions: classes vs timeline vs room. We will stick with the former and do 2 dimensions for now. The latter will likely be the next article in this series.

## The Problem

You need a one-click application to schedule university classes against a single classroom. This classes have differing lengths and may "repeat" throughout the week. Here are the classes:

* Psych 101 (1 hour, 2 sessions/week)
* English 101 (1.5 hours, 2 sessions/week)
* Math 300 (1.5 hours, 2 sessions/week)
* Psych 300 (3 hours, 1 session/week)
* Calculus I (2 hours, 2 sessions/week)
* Linear Algebra I (2 hours, 3 sessions/week)
* Sociology 101 (1 hour, 2 sessions/week)
* Biology 101 (1 hour, 2 sessions/week)


Each repetition must start at the same time of day. The day should be broken up in discrete 15 minute increments, and classes can only be scheduled on those increments. In other words, a class can only start on the :00, :15, :30, or :45 of the hour.

The operating week is Monday through Friday. The operatin day is as follows, with a break from 11:30AM to 1:00PM:

* 8:00AM-11:30AM
* 1:00PM-5:00PM

For each day, these classes cannot be scheduled outside these times.

Create a linear/integer programming model that schedules these classes with no overlap and complies with these requirements.

## Laying the Groundwork

The _very_ first thing you should notice about this problem is the discrete "15 minute" blocks. This is not a continuous/linear problem but rather a discrete one, which is how most schedules are built. Imagine that we have created a timeline for the _entire_ week broken up in 15 minute blocks, like this:

![](images/timeline_concept.jpg)

Note that the "..." block is just a placeholder since we do not have enough room to display the 672 blocks for the week (because 7 days * 24 hours * 4 blocks in an hour).

Now let's expand the concept and make the classes a vertical axis against the timeline. Each intersection is a "slot" that can be 1 or 0. This binary will serve to indicate whether or not that "slot" is the start time for the first recurrence of that class. We will set them all to 0 for now.

![](images/grid_concept.jpg)

This grid is crucial to thinking about this problem logically. It will make an effective visual aid because mathematical constraints will focus on regions within the grid.

On the Kotlin side, let's get our core parameters set up. We are going to take advantage of Java 8's great LocalDate/LocalTime API to make calendar work easier. 


```kotlin
import java.time.LocalDate
import java.time.LocalTime


// Any Monday through Friday date range will work
val operatingDates = LocalDate.of(2017,10,16)..LocalDate.of(2017,10,20)
val operatingDay = LocalTime.of(8,0)..LocalTime.of(17,0)


val breaks = listOf<ClosedRange<LocalTime>>(
        LocalTime.of(11,30)..LocalTime.of(13,0)
)


// classes
val scheduledClasses = listOf(
        ScheduledClass(id=1, name="Psych 101",hoursLength=1.0, repetitions=2),
        ScheduledClass(id=2, name="English 101", hoursLength=1.5, repetitions=3),
        ScheduledClass(id=3, name="Math 300", hoursLength=1.5, repetitions=2),
        ScheduledClass(id=4, name="Psych 300", hoursLength=3.0, repetitions=1),
        ScheduledClass(id=5, name="Calculus I", hoursLength=2.0, repetitions=2),
        ScheduledClass(id=6, name="Linear Algebra I", hoursLength=2.0, repetitions=3),
        ScheduledClass(id=7, name="Sociology 101", hoursLength=1.0, repetitions=2),
        ScheduledClass(id=8, name="Biology 101", hoursLength=1.0, repetitions=2)
)


data class ScheduledClass(val id: Int,
                          val name: String,
                          val hoursLength: Double,
                          val repetitions: Int,
                          val repetitionGapDays: Int = 2)
```
