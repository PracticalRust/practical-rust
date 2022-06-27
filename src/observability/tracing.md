# Tracing

Tracing is both a specific kind of observability and a [Rust crate][tracing-crate] that enables it.

[tracing-crate]: https://docs.rs/tracing/latest/tracing/

## Quick Links

- [tracing][tracing-docs]: the core emitter library. Use this to output events for subscribers to collect.
- [tracing-subscriber]: the core collector library. Use this to actually serialize the events and spans you generate. `tracing-subscriber` has a couple default subscribers to check out. It also defines the necessary traits needed to implement your own.
- [tokio's tracing intro](https://tokio.rs/tokio/topics/tracing). (`tracing` is a part of the broader `tokio` project.)

[tracing-subscriber]: https://docs.rs/tracing-subscriber/latest/tracing_subscriber/

## Tracing: the Practice

Tracing is all about the observability of *per-event data* (~~stolen~~ borrowed from [Brendan Gregg][gregg]). It's not about measuring the overall memory usage of your process over time. It's not about finding the calculating your egress costs by measuring bandwidth. It's all about *events*.

Most applications can be understood as a series of discrete events. Often these events are logically independent. Any request-reply application is trivially event-based. The events are the request-reply pairs.

But even something less obviously event-based can still benefit from drawing certain boundaries to encapsulate "events'. Let's say our application is training a neural network. I would anticipate eventually wanting insight into the training process. What sort of "event data" would be helpful here? I might choose individual trainig epochs as the beginnings of new events. The events would then last until the next epoch, and they would contain summary statistics about the performance of that mini training run and the parameters used.

Of course, not all useful information is found in event data. High-level overviews of system state are still critically important. (Ever run out of space because you forgot to turn on log-rotation? Me neither.) But when it's time to dive into the nitty gritty of what's been happening on your system, event traces will prove invaluable.

[gregg]: https://brendangregg.com/blog/2019-01-01/learn-ebpf-tracing.html


## Tracing: the Crate

> tracing is a framework for instrumenting Rust programs to collect structured, event-based diagnostic information.
> 
> ([The `tracing` docs][tracing-docs])

That about sums it up. Let's unpack some of the key terms.

- **framework**: `tracing` is more than just a crate. It's a set of interfaces for plugging into shared tracing infrastructure. With just `tracing` you can't even see the logs you're producing. The main goal for `tracing` is to give library authors a small dependency that allows them to:
  1. Choose what information is relevant in an event.
  2. Conform to a standard interface that "subscribers" (event collectors) use to collect data.
- **instrumenting**: refers to modifying code to enable the emission of relevant information
- **structured**: machine-parseable, i.e. queryable. It's easy for basic logging to only be amenable to human analysis. For tracing to be effective, the collected information should be consumable by a machine and ready for analytical processing (often by submitting the logs to an OLAP database).

`tracing` itself is pretty bare-bones. It's inteded for use by both library and executable authors, so it's good that it only requires users to bring in a small dependency. `tracing` generally expects that users will either depend on or write other crates to provide the exact suite of functionality they want.

[tracing-docs]: https://docs.rs/tracing/0.1.35/tracing/index.html

### Events and Spans

`tracing` differentiates between "events" and "spans". "Events" are likely a familiar concept; they generally map well to "logs" that we've all seen before. Events generally have a timestamp (they occur at a single instant), some structured data, and a message meant for human consumption.

However, events often fail to capture the relationship that they have *with each other*. During the course of a request-reply session, there can be hundreds or thousands of events. Let's make some up:

```json
{"ts": 1655342020100, "method": "GET", "path": "/"}
{"ts": 1655342020200, "found_user_header": true, "user": "steve3"}
{"ts": 1655342020300, "read": "public/index.html"}
{"ts": 1655342020400, "status_code": 200}
```

These are all individual events, but there's no *structure* that relates them. Sure, we could throw a `request_id` in each of them so that it's possible to correlate them, but why force ourselves into that extra step of analysis? We know at the time of the request that all of these pieces of information are deeply related. Let's encode that into the structure of our events.

> Spans represent periods of time in which a program was executing in a particular context. ([tracing docs][tracing-docs])

A set of related events occurring over time is a "span". Conceptually, a span with all the information from the above example might look like this:

```json
{
    "request_method": "GET",
    "request_path": "/",
    "begin_ts": 1655342020100,
    "end_ts": 1655342020400,
    "events": [
        {"ts": 1655342020200, "found_user_header": true, "user": "steve3"},
        {"ts": 1655342020300, "read": "public/index.html"},
        {"ts": 1655342020400, "status_code": 200}
    ]
}
```

Alomst any analysis of this applications' emitted logs will benefit from the added structure.


