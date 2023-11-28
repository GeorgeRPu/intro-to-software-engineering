---
jupytext:
  formats: md:myst
  text_representation:
    extension: .md
    format_name: myst
kernelspec:
  display_name: Python 3
  language: python
  name: python3
---

# Observability

**Observability** measures how well you can understand the internal state of a sytem from the outputs it emits. Normally, a software system has limited inputs and outputs. For example, a service has API requests as its inputs and responses as its outputs. These responses are not enough to understand the internal state of any complex system. For example, if it takes a long time for a service to respond to a request, it could be because the service has too much traffic, a bug was introduced from a recent code change, or critical hardware failed. From a single response, there is no way of to know if the issue has occurred in the past or this is the first time it has happened.

We must **instrument** our software to emit additional outputs or signals that grant more information about the internal state. **Instrumentation** is the practice of inserting signal producing code into applications. There are 4 types of signals that engineers can instrument their software to emit: logs, metrics, traces, and profiles.

## Logs

**Logs** are human readable messages emitted by a system. Each new log message is appended to existing logs. **Log levels** are used to categorize log messages by importance. Log levels vary by logging library, but the popular [Log4j library](https://logging.apache.org/log4j/2.x/) has 6 log levels, listed below in order of increasing importance.

*Table 1. Log4j log levels in increasing severity.*

| Log Level | Description                                                  |
| --------- | ------------------------------------------------------------ |
| Trace     | Used for fine grain debugging.                               |
| Debug     | Used for debugging.                                          |
| Info      | Used for informational messages.                             |
| Warning   | Used for messages that are not errors, but may indicate an issue. |
| Error     | Used for errors that may allow the application to continue running. |
| Fatal     | Used for errors that will cause the application to abort.    |

Most logging libraries have similar log levels. Log messages with a log level below the configured **logger log level** will not be written. This is useful since more messages are helpful for debugging but is overwhemling for production.

```{code-cell}
import logging

def get_logger(name):
    logger = logging.getLogger(name)
    handler = logging.StreamHandler()
    formatter = logging.Formatter('%(name)s %(asctime)s %(levelname)s %(message)s')
    handler.setFormatter(formatter)
    logger.addHandler(handler)

    return logger

logger = get_logger('my_logger')
logger.setLevel(logging.INFO)
logger.debug('This message will NOT be written.')
logger.info('This message will be written.')
```

There are 2 types of messages that logging software can emit.

Unstructured logs
: The message is a line of text. Usually, the line of text is a sentence that describes an event that has occurred. Best practice is to include the logger name, timestamp, log level, and other metadata in the log message.

```
2023-11-26T03:02:15,015Z INFO Hello, world! Requested by user 123.
```

Structured logs
: Each record is an object and has a format like JSON or XML. Rather than put the timestamp, log level, and other metadata in the log message, they are put in separate fields. The message itself can be split across multiple fields.

```json
{
  "level": "INFO",
  "message": "Hello World!",
  "timestamp": "2023-11-26T03:02:15.015Z",
  "user_id": 123
}
```

The advantage of structured logs is that they are easier to search. For example, suppose we want to find the stack traces for all NullPointerExceptions that occurred in the past 7 days. For unstructured logs, each logged NullPointerException starts with an ERROR message containing NullPointerException. The stack trace is spread across multiple lines. We need a shell command or script that executes the following 3 steps.

1. Filter all log messages that occur in the past 7 days.
2. Find all log messages that contain the string "NullPointerException".
3. Extract the stack trace from subsequent log messages.

For structured logs, each logged NullPointerException is a JSON object with a `stack_trace` field. Many cloud providers have log query languages such as [CloudWatch Logs Insights](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/AnalyzingLogData.html) and [Google Cloud Log Explorer](https://cloud.google.com/logging/docs/view/building-queries) that support these type of queries. Finding the desired stack traces is a single log query that has 3 parts.

1. Filter by timestamp to the past 7 days.
2. Filter by message to include "NullPointerException".
3. Retrieve the `stack_trace` field.

### What to Log

Before deciding what to log, it helps to understand when logs are looked at.
Log are examined almost exclusively to debug issues. Therefore, [errors, failures, and any risky event should be logged](https://devcenter.heroku.com/articles/writing-best-practices-for-application-logs#define-which-events-to-log). The cause of the error or failure, if available, should also be logged. Logs should be as detailed as possible. After an incident occurs, you cannot go back in time to add more logs.

Important events, which are not errors, that occur in a system should be logged. In general, the log level should be Info and above. Info log messages help form a historical record of important events that have occurred in the system. These log messages provide context for future debugging.

These guidelines serve as a minimal starting point. Depending on your use case, you may want to log more. Avoid the temptation to log everything.
This will make your logs unreadable and is expensive.

Sensitive data should NOT be logged.

- Personally identifiable information (PII) such as names, addresses, and government identification numbers.
- Payments information such as credit card number and bank account numbers.
- Security information such as passwords, encryption keys, access tokens, and other secrets.

Leaks of sensitive data will result in lost trust with customers and potentially financial penalties or legal action. Due to the damage that leaks of sensitive data can cause, it has a higher standard for transmission and storage. If your logs contains sensitive data, now the entire logging system must meet this higher standard. Logs need to be encrypted and access restricted which adds friction. This is antithetical to the goal of aiding debugging. Better to not log sensitive data at all.

:::{important}
Keep your logging software up to date. The [Log4Shell vulnerability](https://en.wikipedia.org/wiki/Log4Shell) in December 2021 has made it clear that logging libraries potential surfaces for attackers to exploit.
:::

### Instrumentation

How logging is instrumented depends on the programming language and the logging library. Some logging libraries allow the logger to be constructed with a single line of code. For example, the [`@Log4` annotation](https://projectlombok.org/features/log) in [Lombok](https://projectlombok.org) will create a logger in the variable `log` for the class. In Python, [the standard docs](https://docs.python.org/3/howto/logging.html#advanced-logging-tutorial) recommend to create a logger at the top of each module using the below line.

```{code-cell}
logger = logging.getLogger(__name__)
logger
```

The logger name should reflect the class or module hierarchy and include the class or module name. The name change makes it easier to identify the source of the log message. If possible, separate logs into different streams. For example, as a service processes each request, the logs associated with that request should be written to a separate stream. This makes it easier to follow the logs for a single rquest than the alternative where logs from all requests are interleaved.

Logs should be continuously streamed over network to a centralized **log aggregator**, popularly in the cloud. This includes not just application logs but [operating system logs, server logs, load balancer logs, and other logs from the infrastructure around your application](https://www.splunk.com/en_us/blog/learn/log-aggregation.html). The log aggregator persists logs outside of the machine that generates them. This is important to avoid losing logs if the machine crashes. The log aggregator is a single place to search and analyze logs. Log aggregators often index the logs to speed up searches compared to simply `grep`ing log files.[^b]

Logs should not be retained infinitely. Because logs have primarily string content, they are expensive to store and search. Common retention periods are on the order of months. If you need information about a system from further back, consider using the next signal type.

## Metrics

**Metrics** are numerical measurements of a system. Each metric data point (or datum) is a tuple consisting of the metric name, the numerical value, the units, a timestamp, and a set of key-value pairs called **dimensions** or **labels**[^a]. Depending on the observability system, [data points may have extra fields](https://docs.aws.amazon.com/AmazonCloudWatch/latest/APIReference/API_MetricDatum.html).

```{code-cell}
from datetime import datetime
from typing import Dict

class DataPoint:
    metric_name: str
    metric_value: float  # or int if the metric is a count
    unit: str
    timestamp: datetime
    dimensions: Dict[str, str]
```

Together, these datapoints form a **time series** $x = (x_1, x_2, \dots, x_n)$. The time series is not necessarily equally spaced. For example, a duration metric that measures the time to respond for a service will have a time series with a datapoint for each request. Depending on the traffic, the time between datapoints will vary.

Retaining the individual datapoints is expensive and impractical to use. These raw time series can be aggregated using **statistics**. The statistic divides time into intervals of length $T$ and computes a single values based on the datapoints that occur in each interval. The final result is a time series with equally spaced datapoints.

Average (or Mean)
: For each time interval interval, average the values of all datapoints. For measurement style metrics, this statistic usually makes the most sense. For example, if the metric is the disk usage of a database, using an average statistic over a 5 minute period shows the average disk usage across the 5 minutes.

Max
: For each time interval, take the maximum value of all datapoints.

Min
: For each time interval, take the minimum value of all datapoints.

Percentile
: For each time interval, take the $a$th percentile of all datapoints. Often this is abbreviated as p$a$. For example, p99 is the 99th percentile.

  - p0 is the minimum.
  - p50 is the median.
  - p100 is the maximum.

  Percentile is useful for capturing the tails of a distribution. For example, a useful statistic for services is the p99 of latency. This statistic shows the worst latency 1% of clients experience.

Sample Count
: For each time interval, count the number of datapoints. This statistic is useful to count the number of events that have occurred from a measurement type metric. For example, if the metric is the response latency, the sample count statistic over a 5 minute period is the number of requests that were made every 5 minutes.

Sum
: For each time interval, sum the values of all datapoints. For count style metrics, this statistic usually makes the most sense. For example, if the metric is the number of errors, using a sum statistic over a 5 minute period shows the total number of errors every 5 minutes.

There are many more possible statistics. For example, [AWS CloudWatch has 12 statistics](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Statistics-definitions.html). Below you can see an example of a time series with 100 datapoints and the average, max, median, and min statistics.

```{code-cell}
%config InlineBackend.figure_format = 'retina'
import matplotlib.pyplot as plt
import numpy as np

np.random.seed(0)
x = 100 * np.random.uniform(0, 1, 100)
periods = np.arange(0, 100, 5)

plt.plot(x, label='Data', alpha=0.5)
plt.plot(periods, np.mean(x.reshape(-1, 5), axis=1), label='Average')
plt.plot(periods, np.max(x.reshape(-1, 5), axis=1), label='Max')
plt.plot(periods, np.median(x.reshape(-1, 5), axis=1), label='Median')
plt.plot(periods, np.min(x.reshape(-1, 5), axis=1), label='Min')
plt.xlabel('Time (min)')
plt.ylabel('CPU Utilization (%)')
plt.legend(loc='upper right')
```

### What to Measure

The following metrics should be measured for all services.

- **Dependencies**. If a service $A$ depends on another service $B$, you should measure the inputs and outputs $B$ receives from and sends to $A$. Dependencies can also be databases, queues, and caches. Some common metrics are given below.

  - Number of requests
  - Number of errors
  - Latency

  It can be helpful to add these metrics

- **Resources**. Aim to measure the utilization of all compute, memory, storage, and other resources as percentages of the total capacity.
  It is important to use percentages since the total capacity may be elastic. Depending on the programming language, there may be additional resources that need to be measured, such as heap space for Java or goroutines for Go. Some common metrics are given below.

  - CPU utilization
  - Memory utilization
  - Disk utilization
  - Network utilization

- **Served Traffic**. Just as you should measure the inputs and outputs your service sends to and receives from its dependencies, you should measure the inputs and outputs your service receives from and sends to its clients. This is especially important for services that are exposed to the public. The common metrics are the same as those for dependencies.

These guidelines serve as a minimal starting point. Depending on your use case, you may want to measure more. In particular, metrics are useful for tracking long term trends. For example, you may want to measure the number of users that sign up for your service each day.

:::{seealso}
Further reading: [The Four Golden Signals](https://sre.google/sre-book/monitoring-distributed-systems/#xref_monitoring_golden-signals)
:::

### Instrumentation

How the metrics are instrumented depends on the programming language and the metrics library. Ideally, the metrics library should allow engineers to emit metrics with a single line of code. In languages with annotations or decorators, metrics libraries can provide annotations or decorators that can be applied to functions to emit metrics at the end of the function. For example, the [Python client](https://prometheus.github.io/client_python/) of [Prometheus](https://prometheus.io/), an open source metrics library, has decorators to time functions and emit the duration as a metric.

```{code-cell}
from prometheus_client import Summary
s = Summary('request_latency_seconds', 'The time it takes to respond to a request in seconds.')

@s.time()
def f():
    pass

f()
s.collect()[0]
```

Like with logs, metrics should be exported to an external **metrics collector**. Often this is done by running a **metrics agent** process on the machine that transmits metrics over a network to the metrics collector. The metrics collector persists metrics in a database, outside the machine that generated the metrics, and indexes them for fast search. There are 2 ways metrics can be exported.

Pull
: The metrics collector makes a request, usually HTTP, to the metrics agent to collect metrics. The metrics agent replies with the metrics. An example of a pull based system is Prometheus.

Push
: The metrics agent sends metrics, usually over UDP, to the metrics collector. An example of push based system is [Graphite](https://graphiteapp.org/).

Pull based systems are easier to control and debug since there is a clear distinction between the metrics collector and the metrics agent. Push based systems ensure that metrics are always sent even for short lived processes. Push based system also have an easier time broadcasting metrics to multiple collectors, although it becomes harder to manage metric collection capacity.

:::{seealso}
Further reading: [Push Vs. Pull In Monitoring Systems](https://giedrius.blog/2019/05/11/push-vs-pull-in-monitoring-systems/)
:::

Metrics should be retained for years. Metrics take up less space than logs and so do not need to be deleted as often. Retaining metrics for years allows you to analyze long term trends. One trick is to lose grainularity over time. For example,

- up to 1 minute period metrics for the past 1 day
- up to 5 minute period metrics for the past 1 year
- up to 1 hour period metrics for older than 1 year.

It rare to analyze metrics from more than 1 year ago for minute by minute changes.

## Distributed Tracing



## Continuous Profiling

**Profiling** a type of program analysis that measures the performance of a program.

### What to Profile

### Instrumentation

## When to Use Each Signal Type

[^b]: Log queries are also easier to write than piped grep commands.
[^a]: Observe that the data structure of a metric data point is similar to a structured log. Some observability systems take advantage of this by allowing you to [emit metrics via logs](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Embedded_Metric_Format.html).
