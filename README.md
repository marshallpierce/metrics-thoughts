# Intro

I'd like to propose a re-framing of what [Metrics](https://github.com/dropwizard/metrics) should be and the problems it should attempt to solve. I've written several integrations between Metrics and other libraries/frameworks, used it in several systems, and observed the mailing list and GitHub project for the last few years, and there are some recurring themes of conceptual mismatch between how Metrics currently works and the problems people need solved. The following wall of text is my attempt to construct the "small as possible but no smaller" feature set in the hope that this is what Metrics v4 could look like. I'm trying to avoid too many Metrics v3 biases by starting from some example systems and stating what my assumptions are rather than by describing this as a diff against the Metrics v3 API.

# Goals

- Address core measurement needs for the operation of high volume systems, defined loosely as events that happen often enough on enough servers that some local aggregation of measurements must be performed before emitting them to some storage or reporting system.
- Be compatible with services that are relatively ephemeral: individual service instances come and go all the time.
- Create a *lingua franca* of data types/representations that can be agreed upon by various products throughout the data acquisition, aggregation, and reporting pipeline to eliminate translation burdens and loss of precision.
- Provide a high performance reference implementation for the JVM of the in-process parts of this ecosystem: collecting individual measurements, aggregation, and preparing them for emission to another system (graphite, etc).

# Non-goals

- Tracking events that happen infrequently enough that they can be collected, distributed, etc. individually. If an operation could reasonably be tracked by tossing a message in a distributed queue every time it happens, that's already a problem that can be solved by trivially gluing togther existing infrastructure, so let's focus on other things.
- Health checks. A health check is not a measurement of some aspect of a normally occuring process. It's generally initiated in response to some administrative action, and while it may apply heuristics to various accumulated measurements to respond with a healthy/unhealthy answer, it doesn't meet the definition above of the sort of operation we're concerned with. That doesn't mean health checks are bad, just they're out of scope.
- Exposing things via JMX beyond the core data types referred to above. I've seen some user questions about this, so to be clear: JMX is useful (as long as you keep its limitations in mind), but pressure to improve the JMX capabilities of Metrics v3 to suport richer and possibly user-defined objects is an indication that there is a need to make exposing JMX things easier, not necessarily that it needs to be part of Metrics.

In other words, I'm saying that "calculating the 99.99th percentile" and "is the service healthy" are fundamentally different things, and I don't think they fit well together in the same library, so I'd like to split off the health check stuff into a helper library.

# Motivating example: service processing time

Consider a system that processes incoming events and takes a variable amount of time for each one. If I could have only one measurement of a service, I would want to know how long it spent processing its input, so let's start there. To get something visual to think about, we can model these measurements as a function from time (x axis as the elapsed time since the start of some interval under consideration) to service time (y axis as duration of processing, measured in some arbitrary unit). The function is 0 except at the points when the duration of a unit of work has been measured, at which point the function's output is that duration.

```
    7 -                                                           |
      |                                                           |
    6 -                                               x           |
      |                                                           |
    5 -           x                              x                |
      |                                                           |
 n  4 -                         x                                 |
 o    |                                                           |
 i  3 -                      x                                    |
 t    |                                                           |
 a  2 -                 x               x                         |
 r    |                                                           |
 u  1 -      x                                                    |
 d    |                                                           |
    0 -xxxxxx xxxx xxxxx xxxx xx xxxxxxx xxxxxxxx xxxx xxxxxxxxxxx|
      |---------|--------------------------------------------------
      0         10       time ->
```

This graph represents everything there is to know about the raw measurement data: the magnitude of each measurement, and when it happened. 

If the throughput of the system is low enough that it's not problematic to broadcast these measurements individually (or, equivalently, if the business value of knowing the exact time or duration of each one is high enough to make it worthwhile), then just put each measurement in a message queue and be done with it. The rest of this document is about systems where measurements are too frequent or of too low individual value to broadcast like that. (If you want single-event tracing, perhaps Google's Dapper and its derivatives Zipkin, OpenTracing, etc may be of interest.)

**Assumption 1: Measurements are only useful if we can effectively transport them outside of the service**

Ad-hoc simple measurements, the sort that one might make with some `System.nanoTime()` calls and `println()`, are fine for experimenting on your dev machine. However, that won't take us very far if we have many services running on many machines and we want to collect data from all of them. We should look for ways that allow handing off measurements from one service to another in a structured (and ideally interoperable) fashion for alerting, dashboards, etc.

**Assumption 2: There are too many measurements to transmit them all one by one**

Or, equivalently, the business does not place sufficient value on knowing about each event as it happens to make it worthwhile to building a system with sufficient throughput to be able to monitor each event individually. If measurements are happening too rapidly to transmit each one individually to other systems, then we have no choice but to aggregate them locally for a while and transmit them in batches to ease the strain on monitoring systems.

For our example system, this means we could express tuples of timestamp and measurement (like [[7,1], [12,5], ...] for the example system) in one batch. 

**Assumption 3: It's acceptable to lose time precision within some fixed bound**

We could batch up measurements along with their timestamps, but it seems that in practice knowing the specific timestamp down to the millisecond is generally unimportant as long as we can still say which 1 minute window (or 1 second or 1 hour) it happened in.

This means that we could save space by ditching the timestamp information and simply storing each measurement value along with the number of times that value was measured during the time window. (This is a basic histogram.) For the example system, values 2 and 5 have a count of 2 and all other values have 1. 

### Histogram, so what?

The point of this is to show that we can take the raw, uncompressed data that describes a particular aspect of system behavior in complete fidelity and strip away data we don't care about (timestamps, for instance) and end up with a more efficient data representation that takes into consideration the specific data that system operators tend to care about. There's more we can say about storing this sort of duration data in histograms, but for now let's move on to another example.

# Example #2: counting

Another common task is counting things where the duration of each event isn't particularly important, like "number of users who logged in". We can model this as a function from time to the number of events since the start of the time interval.


```
    7 -                                                           |
      |                                                           |
    6 -                                                 xxxxxxxxxx|
      |                                                           |
    5 -                                      xxxxxxxxxxx          |
      |                                                           |
    4 -                                                           |
      |                                                           |
    3 -                                xxxxxx                     |
 t    |                                                           |
 n  2 -                     xxxxxxxxx                             |
 u    |                                                           |
 o  1 -           xxxxxxxxx                                       |
 c    |                                                           |
    0 -xxxxxxxxxxx                                                |
      |---------|--------------------------------------------------
      0         10       time ->
```

In this case, based the 3 assumptions above, we can collapse this data down to just the number of events since the start of the time window (just a single integral number). In the example above, the compressed form of the data would be simply 6. Tidy!

# Example #3: Measuring attributes of something more complex

Sometimes there are useful bits of data that don't map cleanly into the above situations. Cache hit rate, for instance, is most naturally expressed as a floating point value, but you could use hit and miss counters to express the same thing if you had to. Queue depth is also naturally just a number that changes over time, and it would be nice to express that as-is, but you could get the job done with counters for enqueue and dequeue. And, of course, there are more complex things that can't be easily represented as a combination of counters, like Unix load average. 


```
    7 -                                                           |
      |                                                           |
    6 -                       xxxxxxx                             |
      |                                                           |
    5 -                     xx       x                            |
      |                                                           |
    4 -                 xxxx          x     xx                  xx|
 h    |                                                           |
 t  3 -        xxx     x               x xxx  xxxxxx          xx  |
 p    |                                                           |
 e  2 -x xxxxxx   x   x                 x           xxxx   xxx    |
 d    |                                                           |
    1 - x          xxx                                  xxx       |
 q    |                                                           |
    0 -                                                           |
      |---------|--------------------------------------------------
      0         10       time ->
```

We have a few choices for what to do with this data.

- We can try to compress the data some other way, like using Kolmogorov-Smirnov to match it against a set of different distributions if we have reason to believe it should fit some distribution.
- We can sample periodically and store the result in a histogram, which we already know how to handle (see example #1).
- We can have a degenerate case of the above samples->histogram case with just one sample, in which case we can ditch the histogram data structure and just have one value. (This is basically Metrics v3's `Gauge`.)

The first isn't particularly compelling (especially not for a general-purpose tool). The second is often overkill when we really just want one value. The third we more or less understand since we already have Gauges. However, I suggest that Gauges, at least in their current form, are an antipattern.

# Gauges are awkward

Gauges, as they exist in Metrics v3, are an odd beast: `Gauge<T>` has a `T getValue()` method and that's all. Unlike every other type in Metrics (`Timer`, etc), `Gauge` is a black box. Users don't add data to a `Gauge` and then ask for the `max`; it's just expected to emit something whenever it's interrogated. Gauges exist as a way to allow users to incorporate other data of their choosing into the regular reporting cadence. However, this doesn't work too well in practice. The generic nature of Gauge is misleading. It tempts users to try to do things like `Gauge<MyFancyDataStructure>`, which ends up getting ignored because of course `GraphiteReporter` doesn't know how to serialize that into Graphite-speak. In practice, a Gauge will only really work widely if it emits something numeric. This leads users to try to use [complex multi-Gauge structures](https://github.com/dropwizard/metrics/issues/1016) to get data included with the rest of the Metrics data in reporting, which then has its own problems.

The root problem isn't actually Gauges themselves; it's the way that Metrics handles reporting and scheduling of individual reporting passes. Metrics is fairly black-box in its reporting behavior: once you've integrated your Timers and such, you more or less push the button labeled "do reporting stuff" and then it just happens. This means that users who want to incorporate their own data into Graphite, for example, have to fit into a Gauge because there's no other way to participate. The need for Gauges, and the corresponding complexity we inflict on users, can be avoided if we instead fix the reporting architecture. 

The out-of-the-box functionality to take data from a `Histogram` and emit it to Graphite, etc, is a major part of the value of Metrics, so we don't want to lose that. However, by cracking open the reporting black box a bit, we can keep that ease of use and gain flexibility. (I'll keep using Graphite as an example here, but this applies to any reporter that emits data to some other system, whether it's CSV to stdout or SBE over a socket.) Some reporters have some separation between extracting data from a metric and encoding it into a certain protocol (e.g. `GraphiteSender` and its various implementations, or `GangliaReporter`'s use of `GMetric`), but there isn't a great way for users to interact with those low level abstractions the same way that the rest of Metrics does.

Consider a case like in the multi-Gauge issue linked to above. If we had two changes in the structure of Metrics reporting and scheduling, that use case (and many others) would be able to be handled cleanly. First, we need to expose low-level ways of interacting with reporting backends. Maybe this is simply making it easier to use `GraphiteSender`, or the third-party `GMetric`. Maybe we can even make an SLF4J-like abstraction so that the typical use case of "write a number for a given name" doesn't require any per-protocol customization (though if we go this route, I don't think it should be a requirement for all reporters, just a convenience for ones that it works well for). This would let users write their own data just like Metrics writes the `max` of a `Timer`. To have a reasonable time to write that data, we need another change: inverting [control of scheduling](https://github.com/dropwizard/metrics/pull/1018). If Metrics didn't hide scheduling entirely once initial configuration is done and instead allowed the scheduling cadence to be controlled by user code, it would be trivial for users with advanced needs to use a `ScheduledExecutorService` or similar to execute a snippet of code that invokes all the registered Metrics reporters and then writes all their own custom data to Graphite or whatever. Alternatively, we could allow callbacks to be registered to run at the configured reporting cadence, though I prefer the inverted approach since it can easily be used to create a callback system for cases where that is convenient.

So where does that leave us? I think it may still make sense to have a `LongGauge` and `DoubleGauge` for users that don't need atomicity across complex data structures, etc, but we certainly haven't been well served by `Gauge<T>`. Making a clear path for users to run their own code that writes to the underlying reporting systems when the rest of Metrics does should handle more complex use cases tidily.

# Resetting metrics, aka window duration confusion

In the examples at the start of this document, I hand-waved past this question by simply referring to "time windows". Now let's address that. One of the more frequent user questions is how to reset things, whether in between reporter runs or just in general. (I got enough of these requests that I saw fit to make a special `Histogram` implementation that resets its state when asked for a snapshot -- an ugly hack that doesn't work with multiple Reporters, among other limitations.) This is a specific facet of a broader question: when you ask a `Counter` what its count is, or a `Timer` what its max is, how much time into the past should be considered? Right now, the answer is "since startup". This is initially an attractive way to handle the question because it's easy for ad-hoc human consumption, but it creates several thorny issues. Look at the last few years of the user mailing list for all the struggles people have with this; search for "reset" and similar terms. Here's [one example](https://groups.google.com/forum/#!searchin/metrics-user/reset$20iteration|sort:relevance/metrics-user/90oRLlDTNWU/OnTCnZy5KAAJ).

Consider a `Counter` for "number of requests handled". If you call `getCount()` and get `132`, and then a few minutes later call it again and see `148`, you (the interrogating system) need to have maintained state to get the useful answer of `148 - 132 = 16` requests having happened in the intervening timeframe. Putting this burden of per-node state maintenance on the reporting system doesn't make any sense when it would be a lot cheaper to do so on each individual system, and sure enough, in practice reporting systems don't do this because they shouldn't need to. This creates an [impedance](https://groups.google.com/forum/#!topic/metrics-user/wnK0pHCOJ7s) [mismatch](https://groups.google.com/forum/#!topic/metrics-user/uVLs73pxEtc) with Metrics.

Another problem with "since startup" is how poorly it plays with the varying lifespan of processes. It means that to be able to interpret the data coming out of Metrics, you need to know the uptime of the service, since each datum has an implicit "... since the service came online" attached to it. That means if I have two services in a load balancing pool and one has a request count of 100 and another has 10, I can't tell if anything meaningfully wrong is going on in my load balancing unless I also know the age of each service instance. So, if I say "the counter reads 100", that has absolutely no utility whatsoever unless I also add "and the process is 500s old".

This also makes new processes look radically different from old ones. An old process with lots of data points will have a much more stable 99.9th percentile than a new one with little data where each new datum has a high probability of redefining the 99.9th. This problem leads to awkward solutions like `ExponentiallyDecayingReservoir`, which in my opinion is the wrong choice for almost everyone. It produces fantasy data: numbers that never existed in reality, but are statistical approximations of what data might have existed if the (almost always incorrect) assumption of normality is made. 

I think the correct solution for this is to make each metric have a simple, comprehensible time window policy: as governed by a user-configurable schedule, read the accumulated data in each metric into a per-metric re-usable buffer and reset the metric state. Now, a `Counter` will *always* mean "the count since the last reset". The user knows exactly how long this is, since they configured it, and there are no secret addenda needed to make sense of this number. Old processes behave like new processes, so it won't play havoc with your numbers if you have a fleet of short lived services coming and going constantly. It also makes it possible to easily merge numbers from multiple instances: if 4 services report that their counts were `[12, 16, 8, 11]` for a given time window, then you can trivially add them up to find the total count across every instance for that time window. This is probably the number you *actually want*: the cumulative total across all your services. The same goes for `Histogram` and friends. Note, however, that this means much (all?) of the functionality of `Meter` and `Timer` is now unneeded: we don't need exponentially weighted anything to determine the 5 or 15 minute arrival rates. Instead of relying on math of dubious validity for real-world systems, we can let reporting systems do what they already know how to do: aggregate data like that for us once we get out of their way. This is what time series databases are for, so we should let them do that.

It is a good sign that this way of gathering data will make the `JMXReporter` more complex and all the other reporting pipelines simpler: `JMXReporter` *should* be the odd one out. Anyone who wants to can write a reporter that aggregates the updates it periodically receives to create a "since startup" view, but we should make it easy to do the other style since that's what makes sense for the rest of the ecosystem. Right now, the time horizon used in Metrics makes it better for ad-hoc human inspection and worse for aggregation in reporting systems. This is the wrong tradeoff.

# Reducing allocation

Another problem with the current reporting pipeline is how allocation-heavy it is. All that's needed to fix this is to make each metric write its data into a reusable buffer whenever the schedule needs to update the registered reporters. For a `Counter` this isn't such a big deal; allocating a `long` local is basically free. For a `Histogram`, this will save a lot: `Histogram.getSnapshot()` offers no provisions for re-use, and `Snapshot.getValues()` exposes a `long[]` of every value, even though in practice this is barely used. That's a lot of allocation for every scheduled reporter run! I think we can and should make Metrics zero-allocation in the steady state, or very close to it. Certainly there's no reason that we can't reset and then fill a buffer for each metric at the user-configured interval and let each reporter act on the contents of that buffer.

# Customizability

Currently, Metrics v3 provides two main ways to customize behavior: a pluggable `Clock` used in `Meter`, etc, and a pluggable `Reservoir` used in `Histogram`. It also has some areas that lack customizability: users can't define the `Clock` or `Reservoir` used for newly created metrics that go through the usual `MetricRegistry`-based means, so in practice custom `Clock` and `Reservoir` implementations are awkward to deploy and are relegated to power users. It's also a stated goal to support "custom metrics" for v4.

If we allow users to more easily interact with the scheduling and reporting pipeline, I think that effectively *is* "custom metrics": users can store, process, and aggregate data in whatever means they see fit, but also emit data when the rest of Metrics does. 

### Clocks

If we have robust support for letting users do whatever they want, I think that also gets rid of the need for our custom `Clock` abstraction. The default implementation uses wall clock time. The other `Clock` implementation shipped in Metrics measures elapsed userspace CPU time for its `getTick()`. There may well be cases where users want to measure things by elapsed CPU time, but I don't think that makes sense to be a default supported concept in Metrics. `Clock` is a messy abstraction to begin with: it combines both wall-clock time with an implementation specific `getTick()`. Plus, simple CPU time is not very useful. If you're trying to measure the work done, you really need access to the hardware counters to measure things like the number of L1 cache misses or your overall instructions per cycle. Even if some user wants it, it's definitely a special purpose tool, and it doesn't make sense to switch all time keeping in the system from wall clock time to CPU time, for instance. It would be judiciously deployed for certain measurements, and we already have a way to let them do that unobtrusively: custom metrics, as described above. 

### Histograms

With the new freedom to let users handle their special purpose measurement needs, we can also simplify `Histogram`'s `Reservoir`. There are four implementations included "in the box": `ExponentiallyDecayingReservoir`, `UniformReservoir`, `SlidingWindowReservoir`, and `SlidingTimeWindowReservoir`. 

`SlidingWindowReservoir` keeps the last `k` data points. It is, of course, completely accurate within the limits of the configured size, but to configure it to hold a usefully large number of samples is impractically memory-hungry. 

`SlidingTimeWindowReservoir` keeps all data points accumulated within some configurable time window (as governed by the applicable `Clock` implementation), so at least you could configure this to be the same range as your reporting schedule and not lose data like you might with `SlidingWindowReservoir`, but it will still be impractically large memory-wise. It's also very slow.
 
 `ExponentiallyDecayingReservoir` and `UniformReservoir` do statistical sampling of the incoming data to keep the memory usage under control, but in doing so, they [throw out or mangle](https://groups.google.com/forum/#!topic/metrics-user/yybIwkcb_DQ) typically the most useful data, especially accurate percentiles and max. See [this thread](https://groups.google.com/forum/#!msg/mechanical-sympathy/I4JfZQ1GYi8/ocuzIyC3N9EJ) for more detail on that.

Given that I'm the author of [hdrhistogram-metrics-reservoir](https://bitbucket.org/marshallpierce/hdrhistogram-metrics-reservoir), it probably won't come as a huge surprise for me to advocate that we should get rid of all the other `Reservoir` implementations and only use [HdrHistogram](http://hdrhistogram.org/). HdrHistogram is ideally suited to the sort of measurements that Metrics is built for: it's fast, it's memory efficient, and it's agnostic to the distribution of the underlying data, so it does a great job keeping accurate percentiles on the frequently multi-modal measurements we see in real world systems. In combination with the fixed-time-window approach to scheduling, it enables some very powerful data analysis. If you ask an `ExponentiallyDecayingReservoir` (the default implementation used in v3 `Histogram`, `Timer`, etc) for its 99.9th percentile two times a minute apart, you can't really do anything with those numbers. You can't meaningfully combine or compare them because they represent (1) a measurement that started some effectively unknown time in the past and (2) are a statistical approximation that assumes the underlying data is distributed normally, which is basically always the wrong assumption. You can't answer questions like "what was the 99.9th percentile across both minutes" or even "... just for the last minute". On the other hand, if you take two HdrHistograms, each of which represents a minute's worth of data, you CAN answer those questions in a mathematically valid way. Each HdrHistogram snapshot is both useful on its own and is able to be combined with other ones to yield aggregate info like "99.9th percentile across 1 hour". [Khronus](https://github.com/Searchlight/khronus) is an example of a time series database built to do this sort of stuff with HdrHistograms. There's more to HdrHistogram but I'll draw the line here. The point is that HdrHistogram was purpose built to measure real-world workloads and participate smoothly with reporting and aggregation tools, so we should use it.

# The path to v4

- Split `HealthCheck` into a separate library
- Keep `Counter` and `Histogram`, and probably entirely remove `Meter` and `Timer`
- Remove `Gauge<T>` and perhaps add `LongGauge` and `DoubleGauge` to allow users to quickly express simple cases
- Switch to a "reset on snapshot" approach for all data types
- Invert control of scheduling and reporting to make it easy for users to govern scheduling and add data from their own data types to the reporting flow
- Zero allocation under load once steady state is achieved
- Remove `Clock` abstraction
- Remove `Reservoir` abstraction: `Histogram` is just a HdrHistogram under the hood
