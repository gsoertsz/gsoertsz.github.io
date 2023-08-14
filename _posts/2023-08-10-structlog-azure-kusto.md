---
layout: post
title: "Structlog in Azure for fun and profit"
categories: [python, structlog, azure, log_analytics, kusto]
---

*With the right logging library, and some nifty Kusto queries in Azure Log Analytics, you can leverage structured logging in Azure for fun and profit.*

<!--excerpt-above-->

### Observability and Scale

We still seek to maintain highly available, fault-tolerant and easily evolvable systems, but nowadays we also want high scale, and high elasticity. We leverage containers and orchestrators, along with load-balancing, to distribute our workloads across multiple compute nodes and services. Long gone are the days when we had one or two servers we cared about and just logged in and tailed the console log of the monolithic process we had developed. We are well and truly in the age of ephemeral compute, log aggregators and observability warehouses. 

In that prior world, log messages were textual explanations of what the process was doing. The emphasis was on maximizing the understandability of diagnostic traces for processes that were running, largely to transfer the intuition from the developer to the operations staff, as quickly as possible, in an environment where these were separate people and indeed, organisations. Log traces would read like journals. "Now transforming the provisioning message....", "Provisioning message transformed!". In root cause analysis, an operations team member would scan the logs looking for anomalies. 

The primary complaint in that world was that of excessive I/O. Time spent writing a log message was expensive given what the software had to do. Log levels TRACE, DEBUG, INFO, WARN, ERROR, etc. allowed log messages to be withheld from being written given a configuration value. This was still not ideal, as in most cases, if a process experienced issues when configured to only write ERROR messages, the non-error messages that would help localize the error or help identify normal operation would not be available until the process was reconfigured with a better log level, and then likely restarted. Potentially clearing the fault condition in that case, its easy to see why technologies such as JMX started to appear to allow for [dynamic log level management](https://logging.apache.org/log4j/2.x/manual/jmx.html).

In the current world, there is also less of a place for unstructured descriptive text in log messages, particularly as these are more challenging to aggregate in a manner that allows for easy retrieval and querying. Obviously, there is no universally perfect schema for log messages, but a semi-structured json document is a well supported, highly interoperable and easily ingested and indexed format. This is where structured logging comes in. 

So, scale, elasticity and heterogeniety gives rise to centralised log and metric aggregation. With all logs centralised, context information in log messages becomes critical. These in turn, along with the inherent diversity of log messages, give rise to structured logging with a focus on JSON format.

Log and metric aggregation is a key approach in modern observability disiplines that underpins any modern worthwhile system of scale. Staff seek out a "single pane of glass" for monitoring their system at both the detailed and aggregate level. It is important to get this right, and to do that you need to establish a foundation and approach for logging in software components that allows you to exploit the framework of observability tools at your fingertips. 

So in principle:

**Structured logging is a superior approach in assuring high scale, elastic and distributed services and systems**

Let's explore some tools and approaches that brings this to life.

### Azure Function Apps

A great example of a highly elastic, high scale compute engine is Azure Function Apps.

### Structured Logging and the importance of a logging standard

[Structlog](https://www.structlog.org/en/stable/) is a logging library for python projects. 




### Key Recommendations for fun and profit

* Define a logging guideline, compatible with structured logging
* Bind context variables to narrow the programmatic logging interface and limit diagnostic information propagation
* Define your own logger detached from the root logger
* JSON is a compelling logging output format
* ERROR level only, starting with INFO for handoff-level only, then moving to metric based confirmation of handoff
* Add context dimensions/tags to metrics for high cardinality



