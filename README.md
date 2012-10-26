Bacon.js
========

A small (< 15k minified) reactive programming lib for JavaScript. Written in CoffeeScript.

Inspired largely by RxJs, but includes the `EventStream` and `Property`
concepts from [reactive-bacon](https://github.com/raimohanska/reactive-bacon).

But hey, where's the bacon?

- [CoffeeScript source](https://github.com/raimohanska/bacon.js/blob/master/src/Bacon.coffee)
- [Generated javascript](https://github.com/raimohanska/bacon.js/blob/master/lib/Bacon.js) (see below for building the js yourself)
- [Generated javascript (minified, no asserts)](https://github.com/raimohanska/bacon.js/blob/master/lib/Bacon.min.js)
- [Specs](https://github.com/raimohanska/bacon.js/blob/master/spec/BaconSpec.coffee)
- [Examples](https://github.com/raimohanska/bacon.js/blob/master/examples/examples.html)
- [Diagrams](https://github.com/raimohanska/bacon.js/wiki) for the visual learners

You can also check out my entertaining (LOL), interactive, solid-ass [slideshow](http://raimohanska.github.com/bacon.js-slides/).

And remember to give me feedback on the bacon! Let me know if you've
used it. Tell me how it worked for you. What's missing? What's wrong?
Please contribute!

Intro
=====

The idea of Functional Reactive Programming is quite well described by Conal Elliot at [Stack Overflow](http://stackoverflow.com/questions/1028250/what-is-functional-reactive-programming/1030631#1030631).

Bacon.js is a library for functional reactive programming. Or let's say it's a library for
working with events and dynamic values (which are called Properties in Bacon.js). 

Anyways, you can wrap an event source, 
say "mouse clicks on an element" into an `EventStream` by saying

    var cliks = $("h1").asEventStream("click")
    
Each EventStream represents a stream of events. It is an Observable object, meaning
that you can listen to events in the stream using, for instance, the `onValue` method 
with a callback. Like this:

    cliks.onValue(function() { alert("you clicked the h1 element") })
    
But you can do neater stuff too. The Bacon of bacon.js is in that you can transform, 
filter and combine these streams in a multitude of ways (see API below). The methods `map`,
`filter`, for example, are similar to same functions in functional list programming
(like [Underscore](http://documentcloud.github.com/underscore/)). So, if you say

    function always(value) { return function() { return value } }
    
    var plus = $("#plus").asEventStream("click").map(always(1))
    var minus = $("#minus").asEventStream("click").map(always(-1))
    var both = plus.merge(minus)

.. you'll have a stream that will output the number 1 when the "plus" button is clicked
and another stream outputting -1 when the "minus" button is clicked. The `both` stream will
be a merged stream containing events from both the plus and minus streams. This allows
you to subscribe to both streams with one handler:

    both.onValue(function(val) { /* val will be 1 or -1 */ })

In addition to EventStreams, bacon.js has a thing called `Property`, that is almost like an
EventStream, but has a "current value". So things that change and have a current state are 
Properties, while things that consist of discrete events are EventStreams. You could think
mouse clicks as an EventStream and mouse position as a Property. You can create Properties from
an EventStream with `scan` or `toProperty` methods. So, let's say

    function add(x, y) { return x + y }
    var counter = both.scan(0, add)
    counter.onValue(function(sum) { $("#sum").text(sum) })

The `counter` property will contain the sum of the values in the `both` stream, so it's practically 
a counter that can be increased and decreased using the plus and minus buttons. The `scan` method 
was used here to calculate the "current sum" of events in the `both` stream, by giving a "seed value"
`0` and an "accumulator function" `add`. The scan method creates a property that starts with the given
seed value and on each event in the source stream applies the accumulator function to the current
property value and the new value from the stream.

Properties can be very conventiently used for assigning values and attributes to DOM elements with JQuery.
Here we assign the value of a property as the text of a span element whenever it changes:

    property.assign($("span"), "text")

Hiding and showing the same span depending on the content of the property value is equally straightforward

    function hiddenForEmptyValue(value) { return value == "" ? "hidden" : "visible" }
    property.map(hiddenForEmptyValue).assign($("span"), "css", "visibility")
    
In the example above a property value of "hello" would be mapped to "visible", which in turn would result in Bacon calling

    $("span).css("visibility", "visible")
    
API
===

Creating streams
----------------

`$.asEventStream("click")` creates an EventStream from events on a
jQuery or Zepto.js object

`Bacon.fromPromise(promise)` creates an EventStream from a Promise object such as JQuery Ajax. This stream will contain a single value or an error, followed immediately by stream end.

`Bacon.interval(interval, value)` repeats the single element
indefinitely with the given interval (in milliseconds)

`Bacon.sequentially(interval, values)` creates a stream containing given
values (given as array). Delivered with given interval (in milliseconds)

`Bacon.repeatedly(interval, values)` repeats given elements indefinitely
with given interval (in milliseconds)

`Bacon.fromEventTarget(target, event)` creates an EventStream from events
on a DOM EventTarget or Node.JS EventEmitter object.

`Bacon.fromPoll(interval, f)` polls given function with given interval.
Function should return Events: either Next or End.

`Bacon.later(delay, value)` creates a single-element stream that
produces given value after given delay (milliseconds).

`new Bacon.EventStream(subscribe)` creates an event stream with the given 
subscribe function.

`property.changes()` creates a stream of changes to the Property (see Property API below)

`new Bacon.Bus()` creates a pushable/pluggable stream (see Bus section
below)

Common methods in EventStreams and Properties
---------------------------------------------

`streamOrProperty.map(f)` maps values using given function, returning a new
EventStream. Instead of a function, you can also provide a constant
value. Further, you can use a property extractor string like
".keyCode". So, if f is a string starting with a
dot, the elements will be mapped to the corresponding field/function in the event
value. For instance map(".keyCode") will pluck the keyCode field from
the input values. If keyCode was a function, the result stream would
contain the values returned by the function. The Function Construction
rules below apply here.

`stream.map(property)` maps the stream events to the current value of
the given property. This is equivalent to `property.sampledBy(stream)`.

`streamOrProperty.mapError(f)` maps errors using given function. More
spedifically, feeds the "error" field of the error event to the function
and produces a "Next" event based on the return value. Function
Construction rules apply.

`streamOrProperty.mapEnd(f)` Adds an extra Next event just before End. The value is created
by calling the given function when the source stream ends. Instead of a
function, a static value can be used. You can even omit the argument if

`streamOrProperty.filter(f)` filters values using given predicate function. 
Instead of a function, you can use a constant value (true/false) or a
property extractor string (like ".isValuable") instead. Just like with
`map`, indeed.

`stream.filter(property)` filters a stream based on the value of a
property. Event will be included in output iff the property holds `true`
at the time of the event.

`streamOrProperty.takeWhile(f)` takes while given predicate function holds true

`streamOrProperty.take(n)` takes at most n elements from the stream

`streamOrProperty.takeUntil(stream2)` takes elements from source until a Next event 
appears in the other stream. If other stream ends without value, it is
ignored

`streamOrProperty.skip(n)` skips the first n elements from the stream

`streamOrProperty.do(f)` returns a stream/property where the function f
is executed for each value, before dispatching to subscribers. This is
useful for debugging, but also for stuff like calling the
preventDefault() method for events. In fact, you can
also use a property-extractor string instead of a function, as in
".preventDefault".

`streamOrProperty.not()` returns a stream/property that inverts boolean
values

`streamOrProperty.flatMap(f)` for each element in the source stream, spawn a new
stream using the function `f`. Collect events from each of the spawned
streams into the result stream. This is very similar to selectMany in
RxJs.

stream.flatMap() can be used conveniently with `Bacon.once()` and `Bacon.never()` for converting and filtering at the same time, including only some of the results.

Example - converting strings to integers, skipping empty values:

    stream.flatMap(function(text) {
        return (text != "") ? Bacon.once(parseInt(text)) : Bacon.never()
    })

`streamOrProperty.switch(f)` like flatMap, but instead of including events from
all spawned streams, only includes them from the latest spawned stream.
You can think this as switching from stream to stream


EventStream
-----------

`Bacon.EventStream` a stream of events. See methods below.

`stream.onValue(f)` subscribes a given handler function to event
stream. Function will be called for each new value in the stream. This
is the simplest way to assign a side-effect to a stream. The difference
to the `subscribe` method is that the actual stream values are
received, instead of Event objects. Function Construction rules below
apply here.

`stream.onValues(f)` like onValue, but splits the value (assuming its an
array) as function arguments to `f`

`stream.onEnd(f)` subscribes a callback to stream end. The function will
be called when the stream ends.

`stream.subscribe(f)` subscribes given handler function to
event stream. Function will receive Event objects (see below).
The subscribe() call returns a `unsubscribe` function that you can
call to unsubscribe. You can also unsubscribe by returning
`Bacon.noMore` from the handler function as a reply to an Event.

`stream.scan(seed, f)` scans stream with given seed value and
accumulator function, resulting to a Property. For example, you might
use zero as seed and a "plus" function as the accumulator to create
an "integral" property. Instead of a function, you can also supply a
method name such as ".concat", in which case this method is called on
the accumulator value and the new stream value is used as argument.

Example:

    var plus = function (a,b) { return a + b }
    Bacon.sequentially(1, [1,2,3]).scan(0, plus)

This would result to following elements in the result stream:

    seed value = 0
    0 + 1 = 1
    1 + 2 = 3
    3 + 3 = 6

`stream.skipDuplicates()` drops consecutive equal elements. So,
from [1, 2, 2, 1] you'd get [1, 2, 1]. Uses === operator for equality
checking.

`stream.merge(stream2)` merges two streams into one stream that delivers
events from both

`stream.delay(delay)` delays the stream by given amount of milliseconds

`stream.throttle(delay)` throttles stream by given amount of
milliseconds. This means that event is only emitted after the given
"quiet period".

`stream.bufferWithTime(delay)` buffers stream events with given delay.
The buffer is flushed at most once in the given delay. So, if your input
contains [1,2,3,4,5,6,7], then you might get two events containing [1,2,3,4]
and [5,6,7] respectively, given that the flush occurs between numbers 4 and 5.

`stream.bufferWithCount(count)` buffers stream events with given count.
The buffer is flushed when it contains the given number of elements. So, if
you buffer a stream of [1, 2, 3, 4, 5] with count 2, you'll get output
events with values [1, 2], [3, 4] and [5].

`stream.toProperty(initialValue)` creates a Property based on the
EventStream. You can optionally pass an initial value

`stream.decorateWith(name, property)` decorates stream values (must be
objects) with a new property with the given name and a value taken from
the given Property.

Property
--------

`Bacon.Property` a reactive property. Has the concept of "current value".
You can create a Property from an EventStream by using either toProperty 
or scan method.

`Bacon.constant(x)` creates a constant property with value x.

`property.subscribe(f)` subscribes a handler function to property. If there's
a current value, an `Initial` event will be pushed immediately. `Next` 
event will be pushed on updates and an `End` event in case the source 
EventStream ends.

`property.onValue(f)` similar to eventStream.onValue, except that also
pushes the initial value of the property. See Function Construction
rules below.

`property.onValues(f)` like onValue, but splits the value (assuming its an
array) as function arguments to `f`

`property.onEnd(f)` subscribes a callback to stream end. The function will
be called when the source stream of the property ends.

`property.assign(obj, method, [param...])` calls the method of the given
object with each value of this Property. You can optionally supply
arguments which will be used as the first arguments of the method call.
For instance, if you want to assign your Property to the "disabled"
attribute of a JQuery object, you can do this:

    myProperty.assign($("#my-button"), "attr", "disabled")

A simpler example would be to toggle the visibility of an element based
on a Property:

    myProperty.assign($("#my-button"), "toggle")

Note that the `assign` method is actually just a synonym for `onValue`:)

`property.combine(property2, f)` combines the latest values of the two
properties using a two-arg function. Similarly to `scan`, you can use a
method name instead, so you could do `a.combine(b, ".concat")` for two
properties with array value.

`property.sample(interval)` creates an EventStream by sampling the
property value at given interval (in milliseconds)

`property.sampledBy(stream)` creates an EventStream by sampling the
property value at each event from the given stream. The result
EventStream will contain the property value at each event in the source
stream.

`property.sampledBy(stream, f)` samples the property on stream events.
The result EventStream values will be formed using the given function
`f(propertyValue, streamValue)`. You can use a method name (such as
".concat") instead of a function too.

`property.skipDuplicates()` drops consecutive equal values. So,
from [1, 2, 2, 1] you'd get [1, 2, 1]. Uses === operator for equality
checking.

`property.changes()` returns an EventStream of property value changes.
Returns exactly the same events as the property itself, except any Initial
events. Note that property.changes() does NOT skip duplicate values, use .skipDuplicates() for that.

`property.and(other)` combines properties with the `&&` operator.

`property.or(other)` combines properties with the `||` operator.

Combining multiple streams and properties
-----------------------------------------

`Bacon.combineAsArray(streams)` combines Properties and EventStreams so 
that the result Property will have an array of all property values as its value.
The input array may contain both Properties and EventStreams. In the
latter case, the stream is first converted into a Property and then
combined with the other properties.

`Bacon.combineAsArray(s1, s2, ...) just like above, but with streams
provided as a list of arguments as opposed to a single array.

`Bacon.mergeAll(streams)` merges given array of EventStreams.

`Bacon.combineAll(streams, f)` combines given list of streams/properties
using the given combinator function `f(s1, s2)`. The function is applied in a
fold-like fashion: the first two streams are given to the function
first. Then the result of this operation is combined with the third
stream and so on. In this variant, the combinator function is applied to
the streams themselves, not the stream values.

`Bacon.combineWith(streams, f)` combines given list of streams/properties
using the given combinator function `f(v1, v2)`. In this variant, the
combinator function is used for combining two stream values, not the
streams themselves. This is equivalent to combining the
streams/properties using the combine method like a.combine(b,
f).combine(c.f) etc. For example, you can combine properties containing
arrays into a single array property, with Bacon.combineWith(properties,
".concat").

`Bacon.combineTemplate(template)` combines streams using a template
object. For instance, assuming you've got streams or properties named
`password`, `username`, `firstname` and `lastname`, you can do

    var loginInfo = Bacon.combineTemplate({
        userid: username, 
        passwd: password, 
        name: { first: firstname, last: lastname }})

.. and your new loginInfo property will combine values from all these
streams using that template, whenever any of the streams/properties 
get a new value. For instance, it could yield a value such as

    { userid: "juha", 
      passwd: "easy", 
      name : { first: "juha", last: "paananen" }}

In addition to combining data from streams, you can include constant
values in your templates.

Note that all Bacon.combine* methods produce a Property instead of an EventStream. If you need the result as an EventStream you might want to use property.changes()

    Bacon.combineWith([stream1,stream2], function(v1,v2) {} ).changes()

Function Construction rules
---------------------------

Many methods in Bacon have a single function as their argument. Many of these
actually accept a wider range of different arguments that they use for
constructing the function.

Here are the different forms you can use, with examples. The basic form
would be

`stream.map(f)` maps values using the function f(x)

As an extension to the basic form, you can use partial application:

`stream.map(f, "bacon")` maps values using the function f(x, y), using
"bacon" as the first argument, and stream value as the second argument.

`stream.map(f, "pow", "smack")` maps values using the function f(x, y,
z), using "pow" and "smack" as the first two arguments and stream value
as the third argument.

Then, you can create method calls like this:

`stream.onValue(object, method)` calls the method having the given name,
with stream value as the argument.

`titleText.onValue($("#title"), "text")` which would call the "text" method of the jQuery object matching to the HTML element with the id "title"

`disableButton.onValue($("#send"), "attr", "disabled")` which would call
the attr method of the #send element, with "disabled" as the first
argument. So if your property has the value `true`, it would call
$("#send").attr("disabled", true)

You can call methods or return field values using a "property extractor"
syntax. With this syntax, Bacon checks the type of the field and if it's indeed a method, it calls it. Otherwise it just returns field value. For example:

`stream.map(".length")` would return the value of the "length" field of
stream values. Would make sense for a stream of arrays

`stream.do(".preventDefault")` would call the "preventDefault" method of
stream values. 

`stream.filter(".attr", "disabled").not()` would call `.attr("disabled")` on
stream values and filter by the return value. This would practically
inlude only disabled jQuery elements to the result stream.

If none of the above applies, Bacon will return a constant value. For
instance:

`mouseClicks.map({ isMouseClick: true })` would map all events to the
object `{ isMouseClick: true }`

Methods that support function construction include 
at least `onValue`, `onError`, `onEnd`, `map`, `filter`, `assign`, `takeWhile`, `mapError` and `do`.

Latest value of Property or EventStream
---------------------------------------

`Bacon.latestValue(stream)` will return a function that will return the
latest value from the given stream or property. Notice that the
side-effect of this is that there will be an irremovable subscriber for
the stream that takes care of storing the latest value.

This is not really recommended. Usually you'll do better by using
combinators such as `combine`, `sampledBy` and `combineTemplate`.

Bus
---

Bus is an EventStream that allows you to `push` values into the stream.
It also allows pluggin other streams into the Bus. The Bus practically
merges all plugged-in streams and the values pushed using the `push`
method.

`new Bacon.Bus()` returns a new Bus.

`bus.push(x)` pushes the given value to the stream.

`bus.end()` ends the stream. Sends an End event to all subscribers.
After this call, there'll be no more events to the subscribers. 
Also, the Bus `push` and `plug` methods have no effect.

`bus.error(e)` sends an Error with given message to all subscribers

`bus.plug(stream)` plugs the given stream to the Bus. All events from
the given stream will be delivered to the subscribers of the Bus.

The plug method practically allows you to merge in other streams after
the creation of the Bus. I've found Bus quite useful as an event broadcast
mechanism in the
[Worzone](https://github.com/raimohanska/worzone) game, for instance.

Event
-----

`Bacon.Event` has subclasses `Next`, `End`, `Error` and `Initial`

`Bacon.Next` next value in an EventStream or a Property. Call isNext() to
distinguish a Next event from other events.

`Bacon.End` an end-of-stream event of EventStream or Property. Call isEnd() to
distinguish an End from other events.

`Bacon.Error` an error event. Call isError() to distinguish these events
in your subscriber, or use `onError` to react to error events only.
`errorEvent.error` returns the associated error object (usually string).

`Bacon.Initial` the initial (current) value of a Property. Call isInitial() to
distinguish from other events. Only sent immediately after subscription
to a Property.

Event properties and methods:

`event.value` the value associated with a Next or Initial event

`event.hasValue()` returns true for events of type Initial and Next

`event.isNext()` true for Next events

`event.isInitial()` true for Initial events

`event.isEnd()` true for End events

Errors
------

`Error` events are always passed through all stream combinators. So, even
if you filter all values out, the error events will pass though. If you
use flatMap, the result stream will contain Error events from the source
as well as all the spawned stream.

You can take action on errors by using the `streamOrProperty.onError(f)`
callback.

`streamOrProperty.errors()` returns a stream containing Error events only.
Same as filtering with a function that always returns false.

See also the `mapError()` function above.

An Error does not terminate the stream. The method `streamOrProperty.endOnError()`
returns a stream/property that ends immediately after first error.

Bacon.js doesn't currently generate any Error events itself (except when
converting errors using Bacon.fromPromise). Error
events definitely would be generated by streams derived from IO sources
such as AJAX calls.

Cleaning up
-----------

As described above, a subscriber can signal the loss of interest in new events 
in any of these two ways:

1. Return `Bacon.noMore` from the handler function
2. Call the `dispose()` function that was returned by the `subscribe()`
   call.

Based on my experience on RxJs coding, an actual side-effect subscriber
in application-code never does this. So the business of unsubscribing is
mostly internal business and you can ignore it unless you're working on
a custom stream implementation or a stream combinator. In that case, I
welcome you to contribute your stuff to bacon.js.

EventStream and Property semantics
----------------------------------

The state of an EventStream can be defined as (t, os) where `t` is time
and `os` the list of current subscribers. This state should define the
behavior of the stream in the sense that

1. When a Next event is emitted, the same event is emitted to all subscribers
2. After an event has been emitted, it will never be emitted again, even
if a new subscriber is registered. A new event with the same value may
of course be emitted later.
3. When a new subscriber is registered, it will get exactly the same
events as the other subscriber, after registration. This means that the
stream cannot emit any "initial" events to the new subscriber, unless it
emits them to all of its subscribers.
4. A stream must never emit any other events after End (not even another End)

The rules are deliberately redundant, explaining the constraints from
different perspectives. The contract between an EventStream and its
subscriber is as follows:

1. For each new value, the subscriber function is called. The new
   value is wrapped into a `Next` event.
2. The subscriber function returns a result which is either `Bacon.noMore` or
`Bacon.More`. The `undefined` value is handled like `Bacon.more`.
3. In case of `Bacon.noMore` the source must never call the subscriber again.
4. When the stream ends, the subscriber function will be called with
   and `End` event. The return value of the subscribe function is
   ignored in this case.

A `Property` behaves similarly to an `EventStream` except that 

1. On a call to `subscribe`, it will deliver its current value 
(if any) to the provided subscriber function wrapped into an `Initial`
event.
2. This means that if the Property has previously emitted the value `x`
to its subscribers and that is the latest value emitted, it will deliver
this value to the new subscriber.
3. Property may or may not have a current value to start with. Depends
on how the Property was created.

For RxJs Users
--------------

Bacon.js is quite similar to RxJs, so it should be pretty easy to pick up. The
major difference is that in bacon, there are two distinct kinds of Observables:
the EventStream and the Property. The former is for discrete events while the 
latter is for observable properties that have the concept of "current value".

Also, there are no "cold observables", which
means also that all EventStreams and Properties are consistent among subscribers:
when as event occurs, all subscribers will observe the same event. If you're
experienced with RxJs, you've probably bumped into some wtf's related to cold
observables and inconsistent output from streams constructed using scan and startWith.
None of that will happen with bacon.js.

Error handling is also a bit different: the Error event does not
terminate a stream. So, a stream may contain multiple errors. To me,
this makes more sense than always terminating the stream on error; this
way the application developer has more direct control over error
handling. You can always use `stream.endOnError()` to get a stream
that ends on error!

Examples
========

See [Examples](https://github.com/raimohanska/bacon.js/blob/master/examples/examples.html)

See [Specs](https://github.com/raimohanska/bacon.js/blob/master/spec/BaconSpec.coffee)

See Worzone [demo](http://juhajasatu.com/worzone/) and [source](http://github.com/raimohanska/worzone)

Build
=====

Build the coffeescript source into javascript:

    cake build

Result javascript file will be generated in `lib` directory.

Test
====

Run unit tests:

    npm install&&npm test

Dependencies
============

Runtime: jQuery or Zepto.js (optional; just for jQ/Zepto bindings)
Build/test: node.js, npm, coffeescript

Why Bacon?
==========

Why not RxJs or something else?

- There is no "something else"
- I want my bacon to be open source
- I want good documentation for my bacon
- I think the Observable abstraction is not good enough. It leaves too much room for variations in
behaviour (like hot/cold observables). I feel much more comfortable with EventStream and Property.
- Bacon needs automatic tests. They also serve as documentation.
- Because.

Contribute
==========

Use GitHub issues and Pull Requests.
