---
title: IronCore Observability Data Formats

iep-number: NNNN

creation-date: 2026-07-12

status: implementable|implemented

authors:
- "@peanball"
- "@maybe-another-author"

reviewers:
- "@main-reviewer-1"
- "@main-reviewer-2"
---

# IEP-NNNN: IronCore Observability Data Formats

<!-- toc:start -->
## Table of Contents

- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [Metrics](#metrics)
  - [Logs](#logs)
- [Background Information](#background-information)
  - [Service Discovery for Metrics of Custom Resources](#service-discovery-for-metrics-of-custom-resources)
  - [Structured vs. Unstructured Logging](#structured-vs-unstructured-logging)
  - [Data Augmentation via Deployment](#data-augmentation-via-deployment)
- [Alternatives](#alternatives)
  - [Metrics Format](#metrics-format)
  - [Push-based Metrics Collection](#push-based-metrics-collection)
  - [Parsing Unstructured Logging](#parsing-unstructured-logging)
<!-- toc:end -->

## Summary

This proposal outlines the mechanism of how observability is achieved in
IronCore. At its core, Observability is about the collection and actionable
analysis of measurements and events, i.e. metrics, logs, and traces.

By providing a common transport format for the contained raw information,
implementers know what and how to expose in terms of data about their respective
component.

Possible observability software stacks are not in scope for this proposal, and
are up to the operator of an environment. By using standard interfaces, commonly
available observability stacks can be used directly or with minimal conversion.
This allows operators to build the level of observability that they require.

There are practically no IronCore components that require detailed per-request
latency breakdowns as they are captured by Traces. This proposals focuses on
metrics and logs.

## Motivation

In the current state, metrics and logs are provided on a best effort, sometimes
because they came with the standard template for a component. This proposal
documents best practices and recommendations, so each component can expose
relevant information with a similar naming schema.

This helps operators and administrators that look at the information, create
dashboards and alerting rules to make use of the collected observability data.

Finally, the goal of observability is to set up an environment where the overall
deployment can be monitored at high level with the ability to drill down. It
should not be necessary to log into a box to find logs or other information, as
it has been collected in a more central place already.

### Goals

1. Define transport mechanisms and data format for metrics and logs
2. Define a human and machine readable format that can also be aggregated for
   logs

### Non-Goals

1. Traces, as there are no components in IronCore that benefit from per-request
   latency breakdowns at the moment.
2. Monitoring and observability tool prescription or lock-in based on the
   selected or recommended data format.

## Proposal

The proposal spans the observability data domains of metrics and logs.

### Metrics

For metrics, which are time series of particular states in a system, the
Prometheus wire format and pull-based retrieval are recommended.

Benefits of pull-based retrieval:

1. Metrics are retrieved, and potentially derived, only when necessary.
2. A central entity can control the sampling frequency, keeping data comparable
   across instances. Changing sampling frequency to balance storage requirements
   and information contained in the data can be done centrally and does not
   require changes for each individual component.
3. Connection issues to a particular target can be identified centrally at the
   respective collection point, which allows notifying operators early.

The Prometheus format defines a plaintext data format that is generally
transferred via HTTP from a _metrics endpoint_, where each sample is stored in a
separate line and consists of:

1. Metric name: This is the name of the time series and describes the type of
   data that is store.
2. Tags: A list of additional information that helps identify the origin of the
   particular value, e.g. the originating machine, service, AZ, network segment,
   etc.
3. Value: the measurement value for this particular sample.

```prometheus
# HELP Total number of API requests via HTTP
# TYPE counter
api_http_requests_total{method="POST", handler="/messages"} 12 
^ metric name           ^ tag          ^ tag                ^ value
```

In addition to the metric and its tags, the Prometheus response includes a
description and type information for the sample.

The timestamp of collected data that is stored in a time series database is the
time at collection, which applies to all metrics captured with the same request.

In addition to the tags provided by metrics endpoint, the collector can augment
each sample with additional tags. This keeps the data transferred at each point
minimal. A service does not need to know or write out the availability zone or
other arbitrary segmentation that it is part of. Instead, the collecting central
entity knows this and can extend the sample information for all metrics data
from this particular metrics endpoint.

Metrics for custom resources will be discovered via [HTTP Service Discovery](#service-discovery-for-metrics-of-custom-resources).
Each appropriate controller exposes an HTTP SD endpoint that provides lists of
derived endpoints to be monitored.

### Logs

Logs are written as Structured Logs in JSON line format for production use
cases. Development configurations may use other representations for easier human
readability.

The proposal does not prescribe a logging library, as multiple alternatives
exist. The following fields MUST be present in the output however:

- `timestamp`: the timestamp encoded as [ISO 8601, UTC][iso8601-utc]
- `level`: the severity level of this log message, with the following values:

  | `level` | When to use |
  | ---- | --- |
  | `TRACE` | most detailed information, possibly for each iteration. |
  | `DEBUG` | logs for steps taken during regular operation, e.g. when tasks are completed successfully. |
  | `INFO` | Infrequent information (e.g. connections, session starts) or periodic summary information; automatic recovery of anomalous conditions |
  | `WARN` | Anomalous conditions that may require operator intervention but are generally recoverable or inconsequential for continued operation of the whole process. |
  | `ERROR` | Anomalous conditions that require operator involvement and cannot be auto-resolved by repeat attempts. |
  | `FATAL` | Incorrect configuration that will not allow the process to start. |

- `message`: A human readable message indicating the reason for this log line.

Operators will generally select a level, from which to log, e.g. `INFO` will not
include `DEBUG` or `TRACE` logs, only the more severe levels.

Other fields can include:

- `process`: name of the process or component, e.g. `metalbond`
- `module`: functionality within a process, e.g. a particular control loop.

Further fields to identify to origin can be inserted as part of the log
forwarding mechanism via configuration. Explicitly output the information as
part of a process' logs that is easily known by the process. There is no need to
introduce configuration that is passed through only for the sake of logging.

## Background Information

In this section, topics are explored further, providing background on the
rationale for the recommendation in the proposal.

### Service Discovery for Metrics of Custom Resources

[Prometheus Operator][prometheus-operator] compatible metrics collection is limited to Kubernetes
standard types for the discovery of metrics endpoints with appropriately named
ports in the `spec` of resources such as `Pod`, `Service`, `Endpoint`, etc.

Custom resources, as used in IronCore, are not part of the automatic discovery,
as Prometheus Operator, and compatible solutions, don't know the types and will
not enumerate monitoring ports there.

There are different approaches for [service discovery via explicit `ScrapeConfig`][promop-scrape-config]:

- Kubernetes: the default, applies to Pod, Node, Endpoint, etc. as mentioned above
- static: a static list of host/port to scrape for metrics
- file: A reference to a file that contains targets
- HTTP: a HTTP endpoint that dynamically provides a list of targets to scrape
- DNS: The DNS record contains information on alternate scrape targets.
  This requires sophisticated and dynamic configuration of DNS entries.

Kubernetes is always active, and HTTP Service Discovery is the recommended and
simplest way for our controllers to provide one URL path that lists dependent
metrics targets. The benefit of going the HTTP route is the common and reusable
approach for security (e.g. TLS configuration, common trusted CA) for monitoring.

### Structured vs. Unstructured Logging

Classic logging in software is often based on string concatenation that is nice
to read for a person, e.g.:

```log
2026-01-02 12:42:44 [EntityProcessor] INFO - Processed 412 entities in 13.2ms
```

Depending on the developer's taste or logging framework defaults, the same
information is represented in a myriad different ways. When debugging, it can be
hard to find the origin of the log line, as it is composed of dynamic content.

The semantic meaning of elements, e.g. the severity level, time stamp and its
date format, the origin of the message and possible semantically relevant bits
encoded in the raw message string, are either lost or require complex parsing.

Structured logging, be it in JSON format or other notation, retains the semantic
meaning of individual elements and makes the message a clear indicator for what
happened. By keeping all the dynamic content as explicit fields, this opens up
the possibility of aggregation and math on these values. Logs become metric-like
in their handling and utility, while still remaining useful for the operator.

```json
{"timestamp": "2026-01-02T12:42:44.000Z", "level": "INFO", "message": "Entities Processed.", "count": 412, "elapsed_time_s": 0.132, "origin": "EntityProcessor"}
```

More complex nesting of data are also possible, and should be used where
necessary and beneficial.

At first this may look unusual, but by naming fields, the semantic meaning is
retained. Finding an instance of `Entities Processed.` in the code also be comes
easier.

Finally, with an appropriate observability system, information contained in logs
can be used for statistical analysis, e.g. how many entities were processed in a
given time. At this point, there is an overlap with the information as exposed
by metrics.

### Data Augmentation via Deployment

IronCore components are generally deployed and discovered via Kubernetes.
Metrics and log collectors can be configured to augment collected data by adding
specific fields that are common to _all_ metrics or logs from a particular
endpoint.

Consequently, such common data that is usually not part of the process emitting
metrics or log will be added in post-processing and does not need to be passed
through to each process for the sake of logs.

Metrics and logs can of course be accessed at the individual sources directly,
e.g. for debugging during bootstrapping or recovery. The context information
injected for a more central collection is then still evident, as the operator
explicitly addressed a particular endpoint.

## Alternatives

This proposal covers different aspects. Each aspect has alternatives that are
described in individual groups.

### Metrics Format

Besides the Prometheus metrics format there are other popular transport formats:

1. [Influx line protocol][influx-line], which provides a more compact ability to
   send multiple values for the same metric and set of tags. Influx line is
   generally used as push-format and contains the timestamp of the sample. It is
   the primary format for InfluxDB, but can be handled and converted by other
   tools such as [Telegraf][telegraf].
2. [Graphite][graphite] only allows a metric name with a value, without tags.
   Accordingly, any information about the origin of the metrics must live in the
   metric name, which can lead to exploding cardinality if stored as-is.

Ultimately, the choice to prefer Prometheus is that this format and tooling are
considered CNCF native and there are many tools that can handle this format
well.

### Push-based Metrics Collection

Prometheus is popularized pull-based metrics collection, where a central
instance defines the frequency and point in time, at which information is
retrieved.

While a push-based mechanism is equally feasible, it is not the default for the
recommended Prometheus compatible solutions.

Drawbacks of push-based collection:

- Each emitter must be configured for a particular sampling frequency. In many
  cases it's preferable to have similar time ranges for comparison and simpler
  dashboard creation. This means that individual settings need to be kept in
  sync. Too frequent sampling rates can lead to large data volumes that consume
  resources unnecessarily.
- When a component goes down, metrics will be missing. It is harder to detect a
  gap in metrics than it is to notice a broken network connection initiated
  centrally.

In a real observability system, there will also be forwarding and aggregation of
information from different sources, which will likely include a push-based
approach. This proposal describes exposing metrics in individual components,
with the recommendation to prepare for a pull-based metrics collection.

For events, where the occurrence time is relevant, logs are the natural
mechanism for transfer. Logs always contain the relevant timestamp at which the
log line event occurred. The collection direction for logs is thus not
prescribed. The recommendation is to use push for logs, as then events will
propagate to the final place where operators will look at them with the least
delay.

### Metrics Service Discovery via dummy `Endpoint`

Instead of HTTP Service Discovery, which introduces an additional endpoint, it
is possible to create `Endpoint` entries that are only used to express metrics
scrape targets. This requires the controller to manage `Endpoint` resources for
all dependent metrics providers and ensure that they are consistent with the
currently available set of resources.

One benefit of this approach is that it is easier to see all targets in one go
as you can query all types for Kubernetes service discovery to get an overview.

### Parsing Unstructured Logging

The alternative to structured logs is extracting information from unstructured
logs via pattern matching, e.g. regular expressions.

`logstash` is one generic tool to parse structured information from unstructured
logs.

This is highly error prone and inefficient. Each change in logs requires
adapting logging rules, or encourages rules that extract a minimum of
information from the logs that is then not more than an aggregated `tail`.

[influx-line]:
https://docs.influxdata.com/influxdb/v2/reference/syntax/line-protocol/
[telegraf]: https://www.influxdata.com/time-series-platform/telegraf/
[graphite]:
https://graphite.readthedocs.io/en/latest/feeding-carbon.html#step-3-understanding-the-graphite-message-format
[iso8601-utc]: https://en.wikipedia.org/wiki/ISO_8601#Coordinated_Universal_Time_(UTC)
[prometheus-operator]: https://github.com/prometheus-operator/prometheus-operator
[promop-scrape-config]: https://prometheus-operator.dev/docs/developer/scrapeconfig/#select-scrapeconfigs-from-namespaces-with-specific-labels
