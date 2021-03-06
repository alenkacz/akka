# Testing Actor Systems

As with any piece of software, automated tests are a very important part of the
development cycle. The actor model presents a different view on how units of
code are delimited and how they interact, which has an influence on how to
perform tests.

Akka comes with a dedicated module `akka-testkit` for supporting tests at
different levels, which fall into two clearly distinct categories:

>
 * Testing isolated pieces of code without involving the actor model, meaning
without multiple threads; this implies completely deterministic behavior
concerning the ordering of events and no concurrency concerns and will be
called **Unit Testing** in the following.
 * Testing (multiple) encapsulated actors including multi-threaded scheduling;
this implies non-deterministic order of events but shielding from
concurrency concerns by the actor model and will be called **Integration
Testing** in the following.

There are of course variations on the granularity of tests in both categories,
where unit testing reaches down to white-box tests and integration testing can
encompass functional tests of complete actor networks. The important
distinction lies in whether concurrency concerns are part of the test or not.
The tools offered are described in detail in the following sections.

@@@ note

Be sure to add the module `akka-testkit` to your dependencies.

@@@

## Synchronous Unit Testing with `TestActorRef`

Testing the business logic inside `Actor` classes can be divided into
two parts: first, each atomic operation must work in isolation, then sequences
of incoming events must be processed correctly, even in the presence of some
possible variability in the ordering of events. The former is the primary use
case for single-threaded unit testing, while the latter can only be verified in
integration tests.

Normally, the `ActorRef` shields the underlying `Actor` instance
from the outside, the only communications channel is the actor's mailbox. This
restriction is an impediment to unit testing, which led to the inception of the
`TestActorRef`. This special type of reference is designed specifically
for test purposes and allows access to the actor in two ways: either by
obtaining a reference to the underlying actor instance, or by invoking or
querying the actor's behaviour (`receive`). Each one warrants its own
section below.

@@@ note

It is highly recommended to stick to traditional behavioural testing (using messaging
to ask the Actor to reply with the state you want to run assertions against),
instead of using `TestActorRef` whenever possible.

@@@

@@@ warning

Due to the synchronous nature of `TestActorRef` it will **not** work with some support
traits that Akka provides as they require asynchronous behaviours to function properly.
Examples of traits that do not mix well with test actor refs are @ref:[PersistentActor](persistence.md#event-sourcing)
and @ref:[AtLeastOnceDelivery](persistence.md#at-least-once-delivery) provided by @ref:[Akka Persistence](persistence.md).

@@@

### Obtaining a Reference to an `Actor`

Having access to the actual `Actor` object allows application of all
traditional unit testing techniques on the contained methods. Obtaining a
reference is done like this:

@@snip [TestKitDocTest.java]($code$/java/jdocs/testkit/TestKitDocTest.java) { #test-actor-ref }

Since `TestActorRef` is generic in the actor type it returns the
underlying actor with its proper static type. From this point on you may bring
any unit testing tool to bear on your actor as usual.

### Testing the Actor's Behavior

When the dispatcher invokes the processing behavior of an actor on a message,
it actually calls `apply` on the current behavior registered for the
actor. This starts out with the return value of the declared `receive`
method, but it may also be changed using `become` and `unbecome` in
response to external messages. All of this contributes to the overall actor
behavior and it does not lend itself to easy testing on the `Actor`
itself. Therefore the `TestActorRef` offers a different mode of
operation to complement the `Actor` testing: it supports all operations
also valid on normal `ActorRef`. Messages sent to the actor are
processed synchronously on the current thread and answers may be sent back as
usual. This trick is made possible by the `CallingThreadDispatcher`
described below (see [CallingThreadDispatcher](#callingthreaddispatcher)); this dispatcher is set
implicitly for any actor instantiated into a `TestActorRef`.

@@snip [TestKitDocTest.java]($code$/java/jdocs/testkit/TestKitDocTest.java) { #test-behavior }

As the `TestActorRef` is a subclass of `LocalActorRef` with a few
special extras, also aspects like supervision and restarting work properly, but
beware that execution is only strictly synchronous as long as all actors
involved use the `CallingThreadDispatcher`. As soon as you add elements
which include more sophisticated scheduling you leave the realm of unit testing
as you then need to think about asynchronicity again (in most cases the problem
will be to wait until the desired effect had a chance to happen).

One more special aspect which is overridden for single-threaded tests is the
`receiveTimeout`, as including that would entail asynchronous queuing of
`ReceiveTimeout` messages, violating the synchronous contract.

@@@ note

To summarize: `TestActorRef` overwrites two fields: it sets the
dispatcher to `CallingThreadDispatcher.global` and it sets the
`receiveTimeout` to None.

@@@

### The Way In-Between: Expecting Exceptions

If you want to test the actor behavior, including hotswapping, but without
involving a dispatcher and without having the `TestActorRef` swallow
any thrown exceptions, then there is another mode available for you: just use
the `receive` method on `TestActorRef`, which will be forwarded to the
underlying actor:

@@snip [TestKitDocTest.java]($code$/java/jdocs/testkit/TestKitDocTest.java) { #test-expecting-exceptions }

### Use Cases

You may of course mix and match both modi operandi of `TestActorRef` as
suits your test needs:

>
 * one common use case is setting up the actor into a specific internal state
before sending the test message
 * another is to verify correct internal state transitions after having sent
the test message

Feel free to experiment with the possibilities, and if you find useful
patterns, don't hesitate to let the Akka forums know about them! Who knows,
common operations might even be worked into nice DSLs.

<a id="async-integration-testing"></a>
## Asynchronous Integration Testing with `TestKit`

When you are reasonably sure that your actor's business logic is correct, the
next step is verifying that it works correctly within its intended environment.
The definition of the environment depends of course very much on the problem at
hand and the level at which you intend to test, ranging for
functional/integration tests to full system tests. The minimal setup consists
of the test procedure, which provides the desired stimuli, the actor under
test, and an actor receiving replies.  Bigger systems replace the actor under
test with a network of actors, apply stimuli at varying injection points and
arrange results to be sent from different emission points, but the basic
principle stays the same in that a single procedure drives the test.

The `TestKit` class contains a collection of tools which makes this
common task easy.

@@snip [TestKitSampleTest.java]($code$/java/jdocs/testkit/TestKitSampleTest.java) { #fullsample }

The `TestKit` contains an actor named `testActor` which is the
entry point for messages to be examined with the various `expectMsg...`
assertions detailed below. The test actor’s reference is obtained using the
`getRef()` method as demonstrated above.  The `testActor` may also
be passed to other actors as usual, usually subscribing it as notification
listener. There is a whole set of examination methods, e.g. receiving all
consecutive messages matching certain criteria, receiving a whole sequence of
fixed messages or classes, receiving nothing for some time, etc.

The ActorSystem passed in to the constructor of TestKit is accessible via the
`getSystem()` method.

@@@ note

Remember to shut down the actor system after the test is finished (also in
case of failure) so that all actors—including the test actor—are stopped.

@@@

### Built-In Assertions

The above mentioned `expectMsgEquals` is not the only method for
formulating assertions concerning received messages, the full set is this:

@@snip [TestKitDocTest.java]($code$/java/jdocs/testkit/TestKitDocTest.java) { #test-expect }

In these examples, the maximum durations you will find mentioned below are left
out, in which case they use the default value from configuration item
`akka.test.single-expect-default` which itself defaults to 3 seconds (or they
obey the innermost enclosing `Within` as detailed [below](#testkit-within)). The full signatures are:

>
 * 
   `public <T> T expectMsgEquals(FiniteDuration max, T msg)`
   The given message object must be received within the specified time; the
object will be returned.
 * 
   `public <T> T expectMsgPF(Duration max, String hint, Function<Object, T> f)`
   Within the given time period, a message must be received and the given
function must be defined for that message; the result from applying
the function to the received message is returned.
 * 
   `public Object expectMsgAnyOf(Duration max, Object... msg)`
   An object must be received within the given time, and it must be equal
(compared with `equals()`) to at least one of the passed reference
objects; the received object will be returned.
 * 
   `public List<Object> expectMsgAllOf(FiniteDuration max, Object... msg)`
   A number of objects matching the size of the supplied object array must be
received within the given time, and for each of the given objects there
must exist at least one among the received ones which equals it (compared
with `equals()`). The full sequence of received objects is returned in
the order received.
 * 
   `public <T> T expectMsgClass(FiniteDuration max, Class<T> c)`
   An object which is an instance of the given `Class` must be received
within the allotted time frame; the object will be returned. Note that this
does a conformance check, if you need the class to be equal you need to
verify that afterwards.
 * 
   `public <T> T expectMsgAnyClassOf(FiniteDuration max, Class<? extends T>... c)`
   An object must be received within the given time, and it must be an
instance of at least one of the supplied `Class` objects; the
received object will be returned. Note that this does a conformance check,
if you need the class to be equal you need to verify that afterwards.
   @@@ note
   
   Because of a limitation in Java’s type system it may be necessary to add
`@SuppressWarnings("unchecked")` when using this method.
   
   @@@
 * 
   `public void expectNoMsg(FiniteDuration max)`
   No message must be received within the given time. This also fails if a
message has been received before calling this method which has not been
removed from the queue using one of the other methods.
 * 
   `List<Object> receiveN(int n, FiniteDuration max)`
   `n` messages must be received within the given time; the received
messages are returned.

In addition to message reception assertions there are also methods which help
with message flows:

`public <T> List<T> receiveWhile(Duration max, Duration idle, Int messages, Function<Object, T> f)`

@@snip [TestKitDocTest.java]($code$/java/jdocs/testkit/TestKitDocTest.java) { #test-receivewhile-full }

Collect messages as long as
* they are matching the given function
* the given time interval is not used up
* the next message is received within the idle timeout
* the number of messages has not yet reached the maximum
All collected messages are returned.

`public void awaitCond(Duration max, Duration interval, Supplier<Boolean> p)`

@@snip [TestKitDocTest.java]($code$/java/jdocs/testkit/TestKitDocTest.java) { #test-awaitCond }

   Poll the given condition every `interval` until it returns `true` or
the `max` duration is used up.

`public void awaitAssert(Duration max, Duration interval, Supplier<Object> a)`

@@snip [TestKitDocTest.java]($code$/java/jdocs/testkit/TestKitDocTest.java) { #test-awaitAssert }

Poll the given assert function every `interval` until it does not throw
an exception or the `max` duration is used up. If the timeout expires the
last exception is thrown.

There are also cases where not all messages sent to the test kit are actually
relevant to the test, but removing them would mean altering the actors under
test. For this purpose it is possible to ignore certain messages:

`public void ignoreMsg(Function<Object, Boolean> f)`
`public void ignoreMsg()`

@@snip [TestKitDocTest.java]($code$/java/jdocs/testkit/TestKitDocTest.java) { #test-ignoreMsg }

### Expecting Log Messages

Since an integration test does not allow to the internal processing of the
participating actors, verifying expected exceptions cannot be done directly.
Instead, use the logging system for this purpose: replacing the normal event
handler with the `TestEventListener` and using an `EventFilter`
allows assertions on log messages, including those which are generated by
exceptions:

@@snip [TestKitDocTest.java]($code$/java/jdocs/testkit/TestKitDocTest.java) { #test-event-filter }

If a number of occurrences is specific—as demonstrated above—then `intercept()`
will block until that number of matching messages have been received or the
timeout configured in `akka.test.filter-leeway` is used up (time starts
counting after the passed-in block of code returns). In case of a timeout the test
fails.

@@@ note

Be sure to exchange the default logger with the
`TestEventListener` in your `application.conf` to enable this
function:

```
akka.loggers = [akka.testkit.TestEventListener]
```

@@@

<a id="testkit-within"></a>
### Timing Assertions

Another important part of functional testing concerns timing: certain events
must not happen immediately (like a timer), others need to happen before a
deadline. Therefore, all examination methods accept an upper time limit within
the positive or negative result must be obtained. Lower time limits need to be
checked external to the examination, which is facilitated by a new construct
for managing time constraints:

@@snip [TestKitDocTest.java]($code$/java/jdocs/testkit/TestKitDocTest.java) { #test-within }

The block in `within` must complete after a @ref:[Duration](common/duration.md) which
is between `min` and `max`, where the former defaults to zero. The
deadline calculated by adding the `max` parameter to the block's start
time is implicitly available within the block to all examination methods, if
you do not specify it, it is inherited from the innermost enclosing
`within` block.

It should be noted that if the last message-receiving assertion of the block is
`expectNoMsg` or `receiveWhile`, the final check of the
`within` is skipped in order to avoid false positives due to wake-up
latencies. This means that while individual contained assertions still use the
maximum time bound, the overall block may take arbitrarily longer in this case.

@@@ note

All times are measured using `System.nanoTime`, meaning that they describe
wall time, not CPU time or system time.

@@@

#### Accounting for Slow Test Systems

The tight timeouts you use during testing on your lightning-fast notebook will
invariably lead to spurious test failures on the heavily loaded Jenkins server
(or similar). To account for this situation, all maximum durations are
internally scaled by a factor taken from the [Configuration](),
`akka.test.timefactor`, which defaults to 1.

You can scale other durations with the same factor by using `dilated` method
in `TestKit`.

@@snip [TestKitDocTest.java]($code$/java/jdocs/testkit/TestKitDocTest.java) { #duration-dilation }

### Using Multiple Probe Actors

When the actors under test are supposed to send various messages to different
destinations, it may be difficult distinguishing the message streams arriving
at the `testActor` when using the `TestKit` as shown until now.
Another approach is to use it for creation of simple probe actors to be
inserted in the message flows. The functionality is best explained using a
small example:

@@snip [TestKitDocTest.java]($code$/java/jdocs/testkit/TestKitDocTest.java) { #test-probe }

This simple test verifies an equally simple Forwarder actor by injecting a
probe as the forwarder’s target.  Another example would be two actors A and B
which collaborate by A sending messages to B. In order to verify this message
flow, a `TestProbe` could be inserted as target of A, using the
forwarding capabilities or auto-pilot described below to include a real B in
the test setup.

If you have many test probes, you can name them to get meaningful actor names
in test logs and assertions:

@@snip [TestKitDocTest.java]($code$/java/jdocs/testkit/TestKitDocTest.java) { #test-probe-with-custom-name }

Probes may also be equipped with custom assertions to make your test code even
more concise and clear:

@@snip [TestKitDocTest.java]($code$/java/jdocs/testkit/TestKitDocTest.java) { #test-special-probe }

You have complete flexibility here in mixing and matching the
`TestKit` facilities with your own checks and choosing an intuitive
name for it. In real life your code will probably be a bit more complicated
than the example given above; just use the power!

@@@ warning

Any message send from a `TestProbe` to another actor which runs on the
CallingThreadDispatcher runs the risk of dead-lock, if that other actor might
also send to this probe. The implementation of `TestProbe.watch` and
`TestProbe.unwatch` will also send a message to the watchee, which
means that it is dangerous to try watching e.g. `TestActorRef` from a
`TestProbe`.

@@@

#### Watching Other Actors from Probes

A `TestKit` can register itself for DeathWatch of any other actor:

@@snip [TestKitDocTest.java]($code$/java/jdocs/testkit/TestKitDocTest.java) { #test-probe-watch }

#### Replying to Messages Received by Probes

The probe stores the sender of the last dequeued message (i.e. after its
`expectMsg*` reception), which may be retrieved using the
`getLastSender()` method. This information can also implicitly be used
for having the probe reply to the last received message:

@@snip [TestKitDocTest.java]($code$/java/jdocs/testkit/TestKitDocTest.java) { #test-probe-reply }

#### Forwarding Messages Received by Probes

The probe can also forward a received message (i.e. after its `expectMsg*`
reception), retaining the original sender:

@@snip [TestKitDocTest.java]($code$/java/jdocs/testkit/TestKitDocTest.java) { #test-probe-forward }

#### Auto-Pilot

Receiving messages in a queue for later inspection is nice, but in order to
keep a test running and verify traces later you can also install an
`AutoPilot` in the participating test probes (actually in any
`TestKit`) which is invoked before enqueueing to the inspection queue.
This code can be used to forward messages, e.g. in a chain `A --> Probe -->
B`, as long as a certain protocol is obeyed.

@@snip [TestKitDocTest.java]($code$/java/jdocs/testkit/TestKitDocTest.java) { #test-auto-pilot }

The `run` method must return the auto-pilot for the next message, wrapped
in an `Option`; setting it to `None` terminates the auto-pilot.

#### Caution about Timing Assertions

The behavior of `within` blocks when using test probes might be perceived
as counter-intuitive: you need to remember that the nicely scoped deadline as
described [above](#testkit-within) is local to each probe. Hence, probes
do not react to each other's deadlines or to the deadline set in an enclosing
`TestKit` instance:

@@snip [TestKitDocTest.java]($code$/java/jdocs/testkit/TestKitDocTest.java) { #test-within-probe }

Here, the `expectMsgEquals` call will use the default timeout.

### Testing parent-child relationships

The parent of an actor is always the actor that created it. At times this leads to
a coupling between the two that may not be straightforward to test.
There are several approaches to improve testability of a child actor that
needs to refer to its parent:

 1. when creating a child, pass an explicit reference to its parent
 2. create the child with a `TestProbe` as parent
 3. create a fabricated parent when testing

Conversely, a parent's binding to its child can be lessened as follows:

 4. when creating a parent, tell the parent how to create its child

For example, the structure of the code you want to test may follow this pattern:

@@snip [ParentChildTest.java]($code$/java/jdocs/testkit/ParentChildTest.java) { #test-example }

#### Introduce child to its parent

The first option is to avoid use of the `context.parent` function and create
a child with a custom parent by passing an explicit reference to its parent instead.

@@snip [ParentChildTest.java]($code$/java/jdocs/testkit/ParentChildTest.java) { #test-dependentchild }

#### Create the child using TestKit

The `TestKit` class can in fact create actors that will run with the test probe as parent.
This will cause any messages the child actor sends to *getContext().getParent()* to
end up in the test probe.

@@snip [ParentChildTest.java]($code$/java/jdocs/testkit/ParentChildTest.java) { #test-TestProbe-parent }

#### Using a fabricated parent

If you prefer to avoid modifying the child constructor you can
create a fabricated parent in your test. This, however, does not enable you to test
the parent actor in isolation.

@@snip [ParentChildTest.java]($code$/java/jdocs/testkit/ParentChildTest.java) { #test-fabricated-parent-creator }

@@snip [ParentChildTest.java]($code$/java/jdocs/testkit/ParentChildTest.java) { #test-fabricated-parent }

#### Externalize child making from the parent

Alternatively, you can tell the parent how to create its child. There are two ways
to do this: by giving it a `Props` object or by giving it a function which takes care of creating the child actor:

@@snip [ParentChildTest.java]($code$/java/jdocs/testkit/ParentChildTest.java) { #test-dependentparent }

@@snip [ParentChildTest.java]($code$/java/jdocs/testkit/ParentChildTest.java) { #test-dependentparent-generic }

Creating the `Actor` is straightforward and the function may look like this in your test code:

@@snip [ParentChildTest.java]($code$/java/jdocs/testkit/ParentChildTest.java) { #child-maker-test }

And like this in your application code:

@@snip [ParentChildTest.java]($code$/java/jdocs/testkit/ParentChildTest.java) { #child-maker-prod }

Which of these methods is the best depends on what is most important to test. The
most generic option is to create the parent actor by passing it a function that is
responsible for the Actor creation, but using TestProbe or having a fabricated parent is often sufficient.

<a id="java-callingthreaddispatcher"></a>
## CallingThreadDispatcher

The `CallingThreadDispatcher` serves good purposes in unit testing, as
described above, but originally it was conceived in order to allow contiguous
stack traces to be generated in case of an error. As this special dispatcher
runs everything which would normally be queued directly on the current thread,
the full history of a message's processing chain is recorded on the call stack,
so long as all intervening actors run on this dispatcher.

### How to use it

Just set the dispatcher as you normally would:

@@snip [TestKitDocTest.java]($code$/java/jdocs/testkit/TestKitDocTest.java) { #calling-thread-dispatcher }

### How it works

When receiving an invocation, the `CallingThreadDispatcher` checks
whether the receiving actor is already active on the current thread. The
simplest example for this situation is an actor which sends a message to
itself. In this case, processing cannot continue immediately as that would
violate the actor model, so the invocation is queued and will be processed when
the active invocation on that actor finishes its processing; thus, it will be
processed on the calling thread, but simply after the actor finishes its
previous work. In the other case, the invocation is simply processed
immediately on the current thread. Futures scheduled via this dispatcher are
also executed immediately.

This scheme makes the `CallingThreadDispatcher` work like a general
purpose dispatcher for any actors which never block on external events.

In the presence of multiple threads it may happen that two invocations of an
actor running on this dispatcher happen on two different threads at the same
time. In this case, both will be processed directly on their respective
threads, where both compete for the actor's lock and the loser has to wait.
Thus, the actor model is left intact, but the price is loss of concurrency due
to limited scheduling. In a sense this is equivalent to traditional mutex style
concurrency.

The other remaining difficulty is correct handling of suspend and resume: when
an actor is suspended, subsequent invocations will be queued in thread-local
queues (the same ones used for queuing in the normal case). The call to
`resume`, however, is done by one specific thread, and all other threads
in the system will probably not be executing this specific actor, which leads
to the problem that the thread-local queues cannot be emptied by their native
threads. Hence, the thread calling `resume` will collect all currently
queued invocations from all threads into its own queue and process them.

### Limitations

@@@ warning

In case the CallingThreadDispatcher is used for top-level actors, but
without going through TestActorRef, then there is a time window during which
the actor is awaiting construction by the user guardian actor. Sending
messages to the actor during this time period will result in them being
enqueued and then executed on the guardian’s thread instead of the caller’s
thread. To avoid this, use TestActorRef.

@@@

If an actor's behavior blocks on a something which would normally be affected
by the calling actor after having sent the message, this will obviously
dead-lock when using this dispatcher. This is a common scenario in actor tests
based on `CountDownLatch` for synchronization:

```scala
val latch = new CountDownLatch(1)
actor ! startWorkAfter(latch)   // actor will call latch.await() before proceeding
doSomeSetupStuff()
latch.countDown()
```

The example would hang indefinitely within the message processing initiated on
the second line and never reach the fourth line, which would unblock it on a
normal dispatcher.

Thus, keep in mind that the `CallingThreadDispatcher` is not a
general-purpose replacement for the normal dispatchers. On the other hand it
may be quite useful to run your actor network on it for testing, because if it
runs without dead-locking chances are very high that it will not dead-lock in
production.

@@@ warning

The above sentence is unfortunately not a strong guarantee, because your
code might directly or indirectly change its behavior when running on a
different dispatcher. If you are looking for a tool to help you debug
dead-locks, the `CallingThreadDispatcher` may help with certain error
scenarios, but keep in mind that it has may give false negatives as well as
false positives.

@@@

### Thread Interruptions

If the CallingThreadDispatcher sees that the current thread has its
`isInterrupted()` flag set when message processing returns, it will throw an
`InterruptedException` after finishing all its processing (i.e. all
messages which need processing as described above are processed before this
happens). As `tell` cannot throw exceptions due to its contract, this
exception will then be caught and logged, and the thread’s interrupted status
will be set again.

If during message processing an `InterruptedException` is thrown then it
will be caught inside the CallingThreadDispatcher’s message handling loop, the
thread’s interrupted flag will be set and processing continues normally.

@@@ note

The summary of these two paragraphs is that if the current thread is
interrupted while doing work under the CallingThreadDispatcher, then that
will result in the `isInterrupted` flag to be `true` when the message
send returns and no `InterruptedException` will be thrown.

@@@

### Benefits

To summarize, these are the features with the `CallingThreadDispatcher`
has to offer:

>
 * Deterministic execution of single-threaded tests while retaining nearly full
actor semantics
 * Full message processing history leading up to the point of failure in
exception stack traces
 * Exclusion of certain classes of dead-lock scenarios

<a id="actor-logging"></a>
## Tracing Actor Invocations

The testing facilities described up to this point were aiming at formulating
assertions about a system’s behavior. If a test fails, it is usually your job
to find the cause, fix it and verify the test again. This process is supported
by debuggers as well as logging, where the Akka toolkit offers the following
options:

 * 
   *Logging of exceptions thrown within Actor instances*
   This is always on; in contrast to the other logging mechanisms, this logs at
`ERROR` level.
 * 
   *Logging of special messages*
   Actors handle certain special messages automatically, e.g. `Kill`,
`PoisonPill`, etc. Tracing of these message invocations is enabled by
the setting `akka.actor.debug.autoreceive`, which enables this on all
actors.
 * 
   *Logging of the actor lifecycle*
   Actor creation, start, restart, monitor start, monitor stop and stop may be traced by
enabling the setting `akka.actor.debug.lifecycle`; this, too, is enabled
uniformly on all actors.

All these messages are logged at `DEBUG` level. To summarize, you can enable
full logging of actor activities using this configuration fragment:

```
akka {
  loglevel = "DEBUG"
  actor {
    debug {
      autoreceive = on
      lifecycle = on
    }
  }
}
```

## Configuration

There are several configuration properties for the TestKit module, please refer
to the @ref:[reference configuration](general/configuration.md#config-akka-testkit).