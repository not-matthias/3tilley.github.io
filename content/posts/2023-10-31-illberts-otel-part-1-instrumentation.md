+++
title = "Illbert's OTel in Rust - Part 1 - Instrumentation"

[taxonomies]
tags = ["OpenTelemetry", "Rust", "Python"]
+++


<!-- For some reason has to be removed in Zola ## Illbert's OTel in Rust - Part 1 - Instrumentation -->

I wanted to write some blog posts on using OpenTelemetry (OTel) in Rust. OpenTelemetry will become the dominant player in the telemetry landscape, as it's supported by the majority of the big cloud players and vendors. My personal view is that it's pretty good, albeit (infinitely?) complex. This isn't helped by how new the standard is, as well as the current immaturity of the Rust libraries as the community iterates towards the best way of doing things.

We'll discuss how OTel came about, and why it's useful for your libraries.

### Prerequisites
It's useful to understand what a log, trace, and metric are (collectively known in OTel as Signals). This can either be from reading about them on the [concepts page](https://opentelemetry.io/docs/concepts/signals/) or from your experience using in your own applications pre-OpenTelemetry. That would be tracing systems like Zipkin / Jaeger, and metrics from Prometheus / StatsD.

To understand the end goal, it might also be worth checking out the [OpenTelemetry demo](https://opentelemetry.io/docs/demo/) for Rust or another language. The demo can also be used to impress/persuade your colleagues and management, other people in your coworking space, and maybe even housemates and partners. If you don't want to download and run the Docker containers yourself you can just check out the [screenshots](https://opentelemetry.io/docs/demo/screenshots/). 

#### Improvement:
If anyone has an example of these demos running live and publicly accessible please let me know and I'll link
# First Fundamental - Vendor Agnostic Instrumentation
I sometimes think it's lost within the complex verbiage of OTel what the goal is. The instrumentation goal is for all library code to have a simple and unambiguous way of injecting observability into their code. The rest is the concern of applications.
## You already know how this works

This should seem familiar, as it's the approach we've landed on in the past for logging. A library doesn't care about Splunk, or the licensing around your ELK stack, it merely instruments what it thinks users might find useful - perhaps under a sensible namespace. The application running that library is then responsible for worrying about:
* How to format the messages
* What granularity of messages are desired (INFO, DEBUG)
* What modules and types of logs are desired
* Where to put the log messages
* Rotating files, pushing to syslog, how to batch messages to send to a logging system, etc

Once a library has logged, its responsibilities are complete. To do anything else would be impolite, if not a straight up bug. Python of course has [quirks](https://docs.python.org/3/howto/logging.html#library-config) to work around, but in general, a well-behaved library might look like the below.
```python
# mylib.py
import logging
logger = logging.getLogger(__name__)

def my_func():
    ...
    logger.debug("About to call external service")
    result = call_external_service(endpoint)
    if result:
        logger.info("Success result from external service")
    else:
        logger.error("Error occurred")
```

Rust is similar, but I've not used for the example due to the general movement away from logs towards tracing.

Note the lack of any vendor code. All of the above is standard library. This is similar in other languages, even if there are multiple vendor-agnostic logging libraries, or they aren't part of the standard library as in [Java](https://www.marcobehler.com/guides/java-logging).

## Outside Logging

Buoyed by the success of logging, but hungry for more insight into their application, developers have reached for more tools to see what's going on. Traces - particularly useful for distributed architectures, and Metrics - particularly useful for dashboards and alerting. Jaeger and Prometheus respectively are great tools for this.

Our example Python library instrumented to provide this insight might look like the below:

```python
# mylib.py
import mylib.jaeger.get_tracer
import logging
import prometheus_client

# Vanilla logger
logger = logging.getLogger(__name__)

# Jaeger Tracer for spans
jaeger_tracer = mylib.jaeger.get_tracer()

# Prometheus Metric
prom_counter = prometheus_client.Counter(
    "service_call_count",
    "Count of call to the external service",
    ["endpoint", "is_success"]
)

def my_func():
    ...
    logger.debug("About to call external service")
    with jaeger_tracer.start_span("ExternalServiceCall") as span:
        result = call_external_service(endpoint)
        if result:
            logger.info("Success result from external service")
            prom_counter.labels(endpoint, "true").inc()
            span.log_kv({"event": "Successful call"})
        else:
            logger.error("Error occurred")
            span.log_kv({"event": "Failed call"})
            prom_counter.labels(endpoint, "false").inc()
```

This is cool in that we can see lots about the code as it executes but:
* We're bound to Jaeger and Prometheus, which is questionable for 1st party libs but really no good for third party libraries that could be used anywhere
* It all seems rather cluttered, and some of the information feels like it's been repeated
* The information has to be repeated because the signals are separate. They are independently instrumented and configured

#### Historical note
Before the comments start flowing, I should note that the above would have been State of the Art around 2015, and there has been a [lot of effort](https://www.alibabacloud.com/blog/the-evolution-history-of-observable-data-standards-from-opentracing-and-opencensus-to-opentelemetry_599289) since to be more general:
* The Python Jaeger client was been deprecated in favour of OpenTracing
* Prometheus released OpenMetrics to allow other libraries and services to interact with their data
* Google released their metrics and traces rival OpenCensus
* OpenTracing and OpenCensus combined to form OpenTelemetry
* OpenMetrics is still a separate but [experimentally compatible](https://signoz.io/blog/openmetrics-vs-opentelemetry/) project?
I'm glossing over the details because they are historical, and while they improved things they didn't solve the issue

# Instrumenting with OpenTelemetry

With the General Availability release of OpenTelemetry (track more subtle details [here](https://opentelemetry.io/docs/specs/status/)), we can now take the same approach to all signals as was taken for logging. We'll move to Rust, but the same is possible in lots of other languages, (and the client libraries are often more mature). A similar library might look like like this:

```rust
use tracing::{error, event, info_span, instrument, span, Level};  
use opentelemetry_api::{global, KeyValue};  
use std::thread;  
use std::time::Duration;  
  
#[instrument]  
pub fn expensive_work() -> &'static str {  
    let meter = global::meter("mylib");  
    let counter_step_1 = meter.u64_counter("step_1").with_description("Calls to expensive step 1").init();  
    counter_step_1.add(1u64, &[]);  
    span!(Level::INFO, "expensive_step_1")  
        .in_scope(|| thread::sleep(Duration::from_millis(25)));  
    // Short form of the span above  
    let counter_step_2 = meter.u64_counter("step_2").init();  
    info_span!("expensive_step_2")  
        .in_scope(|| {  
            thread::sleep(Duration::from_millis(25));  
            let pid = std::process::id();  
            match pid % 8 {  
                x if (0..=3).contains(&x) => {  
                    event!(Level::INFO, name = "success", result = x);  
                    counter_step_2.add(1, [  
                        KeyValue::new("result", "success")  
                    ].as_ref());  
                    "Service call succeeded"  
                },  
                x => {  
                    // Shorthand for event!(tracing::Level::ERROR, "failure", result=x);  
                    error!(name = "failure", result = x);  
                    counter_step_2.add(1, [  
                        KeyValue::new("result", "failure")  
                    ].as_ref());  
                    "Service call failed"  
                },  
            }  
        })  
}
```

The combination of moving to Rust, and the nature of the OTel beast means this has added a chunk of extra code. However there are now several advantages:
* We've done away with logging entirely
	* A tracing event without any attributes is similar to emitting a log message - and there is a movement within rust to simply replace log calls to tracing calls. You'll note that the TRACE, DEBUG, INFO, WARN, and ERROR levels match those available in the log crate, and there are various options for your downstream users to treat them as such, if they just want standard logging
  * Bear in mind that in Rust, tracing isn't OTel, but it can be configured downstream to act like it. If you just want to do one thing to tidy up your code, migrating `log -> tracing` and thinking in terms of structured logging is probably it
* We can get better detail
	* Using events in conjunction with spans allows the context of that event to be captured. In our example we can tell that the failure occurred within the call to `expensive_step_2`. Obvious from our snippet of code, but it pushes that context out of the codebase, so it can be seen more easily in an ops plane
  * Additionally imagine that call was made in a number of different places, you can easily filter the output to instances where it happened within `run_calculations_for_important_client()` perhaps
* We can add instrumentation easily for our functions
	* While the manual work with the counters and spans was annoying, did you notice `#[instrument]` macro at the top? That opens and closes another span for us automatically. That macro _was_ presumably fiddly boilerplate to get right for the implementer. But since there should now only be one way of instrumenting things, authors are incentivised to provide those utilities, and we can cheaply sprinkle them in our code
* We get auto-instrumentation
	* Related to the above, the OTel vision is that the majority of your code should now be auto-instrumented. Most likely the external service call would be `reqwest`, `sqlx`, or similar. Library and framework authors do the hard work once, everyone else benefits
  * Previously there were myriad extensions in Python just for tracing like `flask-jaeger`, `flask-zipkin`,  `flask-sleuth`, so many that PyPI disabled the ability to search with pip from the command line (maybe). Now by installing the OpenTelemetry extensions, or in some cases even without, you've got a good start on observability out of the box - just like the situation we got to with logging. Anything extra we add interacts with and complements that

# Pitfalls

It's not all plain sailing:
* Tracing async code needs [special care](https://docs.rs/tracing/latest/tracing/struct.Span.html#in-asynchronous-code)
* It's not as easy as logging to get right. With logs you're just working with arbitrary text - typos and all. To correctly aggregate your data at a later stage with structured logging, you need to take care to name things sensibly - and there is a [large section](https://opentelemetry.io/docs/specs/otel/common/) of the OTel docs dedicated to that
* You need to set up exporters and other things to get the value out of the instrumentation. That we'll deal with another time - but, spoiler alert: the app configuration is significantly harder than the library instrumentation - even just to print to stdout
* It's a new standard, and while the APIs are now largely stable, common best practice and availability of helper methods are still evolving
