---
layout: post
title: "Structlog in Azure for fun and profit"
categories: [python, structlog, azure, log_analytics, kusto]
kramdown:
    syntax_highlighter: coderay
---

*Structured logging is the de-facto standard approach to maximizing analytic and diagnostic value from logs, but at what cost? Luckily, between structlog and Azure Data Explorer, we achieve a superior observability posture without loss of software quality.*

<!--excerpt-above-->

### Observability, Scale, Heterogeneity, and the Aggregation of JSON-formatted Log Messages

We still seek to maintain highly available, fault-tolerant and easily evolvable systems, but nowadays we also want high scale, and high elasticity. In the cloud, we leverage containers and orchestrators, along with load-balancing, to distribute our workloads across multiple compute nodes and services. Long gone are the days when we had one or two servers we cared about and just logged in and tailed the console log of the monolithic process we had developed. We are well and truly in the age of ephemeral compute, log aggregators and observability warehouses. Furthermore, it's not controversial to treat logs from apps as data streams, feeding analytics that help support superior observability postures.

Previously, log messages were textual explanations of what the process was doing. The emphasis was on maximizing the understandability of diagnostic traces, largely to transfer the intuition from the developer to the operations staff, as quickly as possible, in an environment where these were separate teams and indeed, organisations. Log traces would read like journals. "Now transforming the provisioning message....", "Provisioning message transformed!". In root cause analysis, an operations team member would scan the logs looking for normal operational patterns, and hence, anomalies. 

The primary complaint in that world was that of excessive I/O. Time and compute spent writing a log message was expensive given what the software had to do. Log levels TRACE, DEBUG, INFO, WARN, ERROR, etc. allowed log messages to be withheld from being written given a configuration. This was still not ideal, as in most cases, if a process experienced issues when configured to only write ERROR messages, the non-error messages that would help localize the error or help identify normal operation would not be available until the process was reconfigured with a log level of finer granularity. Potentially clearing the fault condition in that case, it's easy to see why technologies such as JMX were used to allow for [dynamic log level management](https://logging.apache.org/log4j/2.x/manual/jmx.html). The I/O tradeoff is still a very relevant consideration today, but while log levels aren't going anywhere, the extent to which log levels are an everlasting tyranny, can be revisited now that log messages are thought of more as useful telemetry streams. i.e. do we need more than three levels, and is there any difference of opinion regarding when / where they should be used?

In the current world, we still wish to leverage log messages, no less for diagnostic purposes, but much more for the purpose of improving products. There is of course the well known virtuous growth cycle where more customers lead to better products through data, and we would like logs to be part of that cycle. This is specifically to offset the typical costs incurred in using 3rd party log aggregators, who charge us not only for ingestion, storage AND retrieval, but are also happy to pass on costs of business and costs of goods sold (COGS). The compass needle should point towards log messages being treated as a data asset, and exploited in much the same way as a data analyst may use product data to generate insights that inform the product's roadmap. 

The current world is also less a place for unstructured descriptive text in log messages, particularly as, in a centralized log aggregation context, these are more challenging to aggregate in a manner that allows for easy retrieval and querying. Brute force text scans are inefficent, and regexes are as powerful as they are objectionable. Obviously, there is no universally perfect schema for log messages, but a semi-structured JSON document is a well supported, highly interoperable and easily ingested and indexed format. When based on JSON, structured logging really shines. 

---

*System **scale**, **elasticity** and **heterogeneity** gives rise to high volume, semi-structured, host-extricated log streams, which requires **centralised** log aggregation in an ergonomic, flexible, efficient and cost effective log **aggregation** tool. As this is costly, we seek consistent, context-rich, high cardinality, structured (JSON) log messages to support an observability posture that **maximises the analytic value in log messages**, ideally **without loss of software quality or performance impact**.*

---

So a tension emerges in the pursuit of maximal analytic value from log messages - Software quality vs. Contextual Richness, and while no one would disagree with structured logging being a key element of a sustainable logging framework for modern cloud systems, the approach isn't a silver bullet, nor does it come without implications worth considering. Let's look at these in more detail. 


### Elements of a Good Logging Practice

A good structured logging practice is one that maximises the analytic value of log messages without loss of software quality. It's worthwhile touching on the considerations and practices that should be brought to bear on a logging system to help achieve this.

#### The Logging API, and Context Smear 

Somewhat naively, to the extent that I/O isn't performance impacting, yielding more information about running software usually means logging as much information as possible at the source, then leveraging a powerful query language via the log aggregation tool to slice and dice the log messages as desired. 

While its possible to add numerous context related keys to a log message, this can create an downward pressure on code quality when interacting with the logging API. Context information might include function arguments, function names, request details, thread details, etc. This information needs to be stored globally or passed around so it can be referenced in constructing log messages when interacting with the logging API. 

This obviously isn't so prevalent early in a component's lifecycle, but over time, under the demands of a meaningful, long lasting production workload, the code is enhanced to log better messages for the purposes of improving supportability. Better log messages often requires more context information, referring to variables outside the natural scope around the code of interest. 

I refer to this as "context-smear" - previously well structured purposeful code, impacted by needing to pass context information around for diagnostic purposes.

---

*Context-smear is an emergent property of code in production using a not-so-great logging API.*

---

#### The Log Aggregation Tool, Log Extrication, and the Kitchen Pantry Effect

A log aggregation tool that has an incapable query language, under-performing query engine, or costs too much to store or extract data can defeat your aspirations to yield value from logs. Log analytic tools should be evaluated against:

- costs (retention, extraction, storage, support)
- query performance (query and retrieval latency)
- sovereignty and global regional support
- security, access control
- secure (key) material, and classified data handling

Also, when using a 3rd party log aggregation tool, **Your** logs are being shipped to a **3rd Party**, and they are **Charging** you to access your own data. If you can deal with that, fine, but providers can (and do) pass on their cost of business to you via pricing changes, not to mention throttle you if you impact other customers of theirs. This can create an upward cost pressure, and unless your system's logging telemetry is yielding analytic value enough to offset this cost, **limiting** the degree to which your system **over-subscribes** to a 3rd party log aggregator, is key to maximizing the value logs can yield over time.

Log aggregation tools also end up looking like a kitchen pantry after a while, full of somewhat-used and often altered analytic products, notebooks, visualizations, dashboards etc, authored by most of the team. Users come and go for various purposes, put things in, change things a bit, but very rarely throw things out. I have yet to encounter a tool that doesn't look like this after the first 6 months.

That being said, a good log aggregation tool is pivotal in maximizing the analytic value of logs, and while the best tool is cheap, flexible, performant, secure, and centralized, with some team discipline, this situation can be made manageable. The benefit is that the tool remains easy to use, and doesn't suffer for the inherent entropy associated with open-entry analytic workspaces. Ideally, your log aggregation tool is the same as your OLAP system, consistent with logs being treated as telemetry data. But if we take that further, does that spell the death of log aggregation tools as separate services? If I have an existing OLAP system or data warehouse, which already maximises the analytic value of data, why do I need a 3rd party service?

Interestingly, in Azure in particular, Log Analytics is [built on top of](https://learn.microsoft.com/en-au/azure/azure-monitor/logs/log-analytics-overview#relationship-to-azure-data-explorer) Azure Data Explorer (with some additional extras). While the actual tables being worked on differ, the analytic experience is the same, and the solution applies across both Log Analytics and more generalized time-series data streams. It is also easy in Azure, via a service's diagnostic setting, to forward logs to an event hub, and then consume logs in Azure Synapse Analytics. Certainly the distinction between log analytics tools and data analytics in general in Azure is diminishing, if not blurred. This leads us to the obvious question...

---

*If I'm trying to maximize the analytic value of logs, and I have an existing data warehouse, do I need a separate log aggregation tool?*

---

#### Variation, Team Autonomy, and the Data Steward's Mentality

While you may have a smear-minimizing logging API, and a cost-effective, performant log aggregation tool, inconsistencies in the information being logged, and the logging approach taken by developers across domains of different sizes, can also complicate matters. One team may log certain events at INFO level, others at DEBUG. Some teams may have disabled all but ERROR messages, another may be logging everything down to TRACE level. Inconsistencies in the approach at the source, when observed across all log emitting systems, results in both high coupling between the queries and query sprawl in the log aggregation tool.

From a purely data governance point of view, in recognition that a datawarehouse may contain data from multiple domains, a Data Steward is typically involved in the governance, quality, semantic correctness and vitality of data for a particular domain. Logs are a form of data, and as we seek to maximise its analytic utility, it's not irrelevant to wonder what a Data Steward may think about log messages that lack some form of consistency, control or quality criteria. Unsuprisingly, a Data Steward might reach for a standard, and equivalently, a **logging standard** is a useful instrument in defining a north star as to what, why and how of logging across teams, without constraining teams too greatly with centralized bureacracy. Managing non-compliance and enforcement, is left as an exercise for the reader, however it is worthwhile discussing some key pre-requisities for a logging standard that yields the desired outcome, remains meaningful and relevant, can be adhered to and doesn't become outmoded the moment its promulgated.

##### The Path To Production

Delivery teams use different environments to manage software quality between development and production. Different log levels are relevant in different environments to balance the tradeoff between cost of storage, utility, and the I/O impact of fine-grained logs.

Define a path to production which outlines the set and sequence of environments (and their purposes). At a cursory level, this may be something like:

| Environment | Purpose                     | Log Level | Retention Level |
| ------------|-----------------------------|-----------|-----------------|
| Development | N/A                         | DEBUG     | Short           |
| Testing 1   | Automated Integration Test  | INFO      | Short           |
| Testing 2   | SIT                         | INFO      | Short           |
| Staging     | Performance/UAT             | ERROR     | Short           |
| Production  | N/A                         | INFO      | Long            |


The above approach recognises that non-production logs have higher utility in early stages of development as the focus is on validating correct software behaviour. There may be some utility in non-integrated testing, but given that this won't typically relate to production usage or performance, there is less valuable insight to be had.

##### Exploratory log analytics in development

A primary objective of the logging standard is to help address the quality and consistency of log data. 

It should attempt to set out clear expectations on what good log messages are, and when and how to log them appropriately in a general sense. Assessing the quality of log data really involves attempting to use the data in some form of analysis. At the very least, within the logging standard, and in the language of the log aggregation tool, publish examples of desired log analytic queries that yield value, such as p99 latency, or error rates.

Doing this, in practice, demonstrates and reinforces the motivation and intent for logging, specifically focused on utility. As a guideline, once a developer understands the intention behind the standard, they are able to more effectively balance the twin-objectives of logging: **diagnostic richness**, and **analytic utility**.

Both AWS [Cloudwatch Log Insights](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_QuerySyntax-examples.html) and Azure [Log Analytics](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/get-started-queries), provide powerful tools for log analysis. So it's straightforward to identify some useful queries that demonstrate analytic value from logs, and to express this in the standard.

Log messages generated from live code in reality, even in development, also provides useful inputs to the cost model. As costs are a significant consideration for log aggregation, initial feedback on costs is key to shaping a logging standard, and is also the right unit-vector towards ensuring costs are a primary platform consideration. 

---

*To embed this further, actually execute these queries in development, put the example results in the logging standard, and monitor costs to generate an initial cost model contribution factor.*

---

##### API usage examples

Once you have a language of choice (or languages 😉), you can then select the Logging API you wish to use. Assuming its a good one, put configuration and usage opinions right into the logging standard. 

Clear instructions on how to use and configure a logging API for a particular language helps to reify the logging standard, not only for developers but for code owners and code reviewers. 

##### Iterate and Adapt

The first version of the logging standard should be v0.1, implying it's a pre-release informal version. Once you have a system that adheres to the logging standard, with code in production, you can then do things like:

- Get developer feedback on levels 
- Measure costs and develop a cost model
- Evaluate the ergonomics, hygiene and performance of the log analytics tool
- Evaluate how well the logging approach (supported by the standard), fits into your overall observability regime

No logging standard is perfect and final to begin with, but it can be meaningful and relevant from the beginning. 

### In Practice

To demonstrate some of these considerations, we can deploy an Azure Function App written in python, that uses structlog to do some logging, along with a Log Analytics Workspace to which its logs get sent.

### Azure, Python, Function Apps and the Structlog logging API

[Structlog](https://www.structlog.org/en/stable/) is a structured logging library for python projects. It has a run-of-the-mill logging API (log(...), debug(..), etc) and handler/formatter framework, but also provides good support for single threaded, as well as multi-threaded (flask) apps, and also provides good support for [managing context information](https://www.structlog.org/en/stable/contextvars.html). It provides methods for managing the **structlog** global context variables that we wish to emit as part of our events. Let's examine how this might work in an Azure Function App implementation.

#### Python Function App

The python function app seeks to demonstrate how structlog can be used for structured logging. By way of example, the function app uses the [Open Weather API](https://openweathermap.org/). Function Apps are a good example of host-extricated logs - we would very rarely log into the 'host' of a Function App (although in Azure this is possible)

The Open Weather API is a rich API service, that provides current weather information given a geocoded location (lat, long). Obviously this is a little inconvenient, as it requires that a plain english location (city, state, country etc) needs to be geocoded into coordinates. Conveniently, the Open Weather API provides a geo-coding service as well. The function app, accepts a request containing a city, state, and country, then:

1. Uses the geocoding API to obtain the latitude (lat) and longitude (long) of the the location
2. Uses the weather API to retrieve the current weather for the lat/long acquired in (1)
3. Returns the current weather payload to the user

Because the Weather/Geocoding API requires an API key, the function app is deployed with an Azure KeyVault, and is configured with the secret to read. To use the Azure Key Vault, the Key Vault is configured to enable RBAC authorization, and the System Assigned managed identity for the function app is assigned the `Key Vault Secrets User` role on the Key Vault. This can be seen in the infrastructure template located [here](https://github.com/gsoertsz/azure-function-app-structlog-example/blob/03141827f905f92dc25030a3d746f4b895a30fc4/bicep-templates/functionapp.bicep#L166)

When servicing the request the function app retrieves the Weather API Key, then does the work above to provide the current weather back to the user. Below is an example request to the function app:


```json
%> curl -i -X GET -d '{ "country_code": "xxx", "state": "xxx", "city": "xxxxx" }' https://slg-fn.azurewebsites.net/api/Function\?code\=xxxxxxxxxxxxxxxxxx
HTTP/1.1 200 OK
Content-Type: text/plain; charset=utf-8
Date: Tue, 29 Aug 2023 05:02:16 GMT
Server: Kestrel
Transfer-Encoding: chunked
Request-Context: appId=cid-v1:586e5506-29ed-4d66-a3d3-d3fc0c215814

{
    "coord": {
        "lon":xxxxxx,
        "lat":xxxxxx
    },
    "weather": [
        { 
            "id":804,
            "main":"Clouds",
            "description":"overcast clouds",
            "icon":"04d"
        }
    ],
    "base":"stations",
    "main": {
        "temp":17.19,
        "feels_like":16.77,
        "temp_min":15.53,
        "temp_max":18.23,
        "pressure":1013,
        "humidity":69
    },
    "visibility":10000,
    "wind": {
        "speed":9.26,
        "deg":30
    },
    "clouds": {
        "all":97
    },
    "dt":1693285118,
    "sys": {
        "type":2,
        "id":2008797,
        "country":"xxx",
        "sunrise":1693255648,
        "sunset":1693295739
    },
    "timezone":xxxxx,
    "id":2158177,
    "name":"xxxx",
    "cod":200
}
```

Here is the code for the main handler function of the function app:

```python
import azure.functions as func

from .handlerfunction import Handler
from azure import functions as func
from azure.keyvault.secrets import SecretClient
from azure.identity import ManagedIdentityCredential
from datetime import datetime
import pytz
import os
import structlog

log = structlog.get_logger("function")

def main(req: func.HttpRequest, context: func.Context) -> func.HttpResponse:

    tz = pytz.timezone('Australia/Sydney')
    ingestion_datetime = datetime.now(tz)

    vault_url = os.environ["KEY_VAULT_URL"] if "KEY_VAULT_URL" in os.environ else None
    
    if not vault_url:
        raise Exception("No Key Vault URL")

    managed_identity_credential = ManagedIdentityCredential()
    secret_client = SecretClient(
        vault_url=vault_url,
        credential=managed_identity_credential
    )
    
    try:
        handler = Handler(
            secret_client=secret_client
        )

        return handler.handle(req, context, ingestion_datetime)
    except Exception as e:
        log.error(event="Unable to execute Handler: {}".format(e), exc_info=True, error_code = 1001)
        raise e

```

Here is the code for handler service of the function app:

```python

from azure import functions as func
from azure.keyvault.secrets import SecretClient
from attrs import define
from datetime import datetime
from shared_code.utils import responseutils as responses
import structlog
import os
import requests
import json
from structlog.contextvars import bind_contextvars
from shared_code.utils import loggingutils


log = structlog.get_logger("function")

@define
class Handler(object):
    secret_client: SecretClient

    @loggingutils.log_context(logger_to_use=log)
    def handle(self, request: func.HttpRequest, context: func.Context, ingestion_datetime: datetime) -> func.HttpResponse:
    
        bind_contextvars(**request.headers)
        bind_contextvars(**vars(context))

        log.info(event="Handling request")

        weather_api_key_secret_name = os.environ["WEATHER_API_KEY_NAME"] if "WEATHER_API_KEY_NAME" in os.environ else None

        if not weather_api_key_secret_name:
            raise Exception("No API Key Secret Name")

        weather_api_key_secret = self.secret_client.get_secret(name=weather_api_key_secret_name)

        request_dict = json.loads(request.get_body())
        request_city = request_dict["city"]
        request_country_code = request_dict["country_code"]
        request_state_code = request_dict["state"]

        r_weather = self.city_request_to_weather(
            appid=weather_api_key_secret.value,
            city=request_city,
            state=request_state_code,
            country=request_country_code
        )

        log.info(event="Successfully processed")

        return func.HttpResponse(body=bytes(r_weather.text, "utf-8"), status_code=200)
    
    @loggingutils.log_context(logger_to_use=log)
    def city_request_to_weather(self, appid: str, city: str, state: str, country: str) -> requests.Response:
        query = f"{city},{state},{country}"
        
        geocode_param_dict = {
            "q": query,
            "limit": 1,
            "appid": appid
        }

        r_geocode = requests.get("http://api.openweathermap.org/geo/1.0/direct", params=geocode_param_dict)
        response_dict = json.loads(r_geocode.text)
        lat = response_dict[0]["lat"]
        lon = response_dict[0]["lon"]

        log.debug(event=f"Received geocoding response |{r_geocode.text}|")

        weather_req_params = {
            "lat": lat,
            "lon": lon,
            "units": "metric",
            "appid": appid
        }

        r_weather = requests.get("https://api.openweathermap.org/data/2.5/weather", params=weather_req_params)

        return r_weather
```

Below is the package level `__init__.py` file, containing the logging configuration:

```python

import logging
import structlog
from structlog.stdlib import LoggerFactory

myLogger = logging.getLogger("function")
myLogger.addHandler(logging.StreamHandler())
myLogger.setLevel(logging.DEBUG)

structlog.configure(
    logger_factory=LoggerFactory(),
    processors=[
        structlog.contextvars.merge_contextvars,
        structlog.stdlib.filter_by_level,
        structlog.processors.StackInfoRenderer(),
        structlog.processors.format_exc_info,
        structlog.stdlib.add_log_level,
        structlog.stdlib.add_logger_name,
        structlog.processors.dict_tracebacks,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.JSONRenderer()
    ]
)

```

The configuration above does the following:

- prints stack traces for errors
- Format's timestamps as ISO-8601
- prints logger name and level
- prints the log message in JSON format

This setup will provide a pretty useable structured logging configuration. But there are some noteworthy things happening in the code above worth discussing.

##### Context variable binding

From the previous discussion, avoiding context-smear is key to limiting the impact logging has on software quality and clarity. As can be seen in the code above, structlog offers `bind_contextvars(..)` allowing map-like/hashtable-like structures to be provided and bound to the logging context as key value pairs. These appear as key-valued dimensions when written as structured logs, which when indexed in the log aggregation tool, become column references that can help group/partition/join table structures etc.

##### Standardization of logs with python decorator

We can leverage structlog's context management functions, its general `log` message, and python decorators, to consistently log the entry to and exit from different functions within the implementation.

Leveraging decorators, allows us to mark functions that we are interested in. Once marked with the logging decorator, we can emit "Entering", and "Exiting" events prior to, and after execution respectively, as well as adding function details (name, args, etc) to the context information around logs. Additionally:

- If we do any direct logging within the marked function, we want those logs to come through with the same context information. 
- Once we exit the function we want the additional context (for the nested function) to be removed, and the prior context to be restored for the outer scope (if applicable)
- We want the logger used in the decorator to be customizable with respect to level to avoid surprises

Below is definition of the decorator that does all of this:

```python 
from typing import Any
from structlog.contextvars import bind_contextvars, unbind_contextvars, get_contextvars, reset_contextvars
import structlog
import logging

def log_context(logger_to_use: Any = None, level: int = logging.DEBUG) -> Any:

    def inner_wrapped(func) -> Any:
        def wrap(*args, **kwargs):
            param_dict = {
                "args": args, 
                "function": func.__name__
            }

            tokens = bind_contextvars(**param_dict, **kwargs)

            if logger_to_use:
                logger_to_use.log(level=level, event=f"Entering")

            result = func(*args, **kwargs)

            if logger_to_use:
                logger_to_use.log(level=level, event=f"Exiting")

            reset_contextvars(**tokens)

            return result

        return wrap
    return inner_wrapped
```

This decorator can be used as follows:

```python

from shared_code.utils import loggingutils
import structlog
import logging

log = structlog.get_logger("function")

def test_loggingcontext_annotation():
    logging_context_helper_fn1(str_arg="some string")

    assert True

@loggingutils.log_context(logger_to_use=log)
def logging_context_helper_fn1(str_arg: str) -> str:
    log.info(event="This is the first fn call")
    logging_context_helper_fn2(str_arg=f"{str_arg}, with some more data")
    log.info(event="After the nested call in the first scope")


@loggingutils.log_context(logger_to_use=log, level=logging.INFO)
def logging_context_helper_fn2(str_arg: str) -> str:
    log.info(event="This is the second function call")
    return str_arg
```

For the above functions, executed as tests, let's examine the log messages received

```terminal
2023-09-04 15:09:00 [debug] Entering                       args=() function=logging_context_helper_fn1 str_arg=some string
2023-09-04 15:09:00 [info ] This is the first fn call      args=() function=logging_context_helper_fn1 str_arg=some string
2023-09-04 15:09:00 [info ] Entering                       args=() function=logging_context_helper_fn2 str_arg=some string, with some more data
2023-09-04 15:09:00 [info ] This is the second function call args=() function=logging_context_helper_fn2 str_arg=some string, with some more data
2023-09-04 15:09:00 [info ] Exiting                        args=() function=logging_context_helper_fn2 str_arg=some string, with some more data
2023-09-04 15:09:00 [info ] After the nested call in the first scope args=() function=logging_context_helper_fn1 str_arg=some string
2023-09-04 15:09:00 [debug] Exiting                        args=() function=logging_context_helper_fn1 str_arg=some string
```

In the above scenario, entry/exit logging for `logging_context_helper_fn1` was done at debug level (default), whereas for `logging_context_helper_fn2`, entry/exit logging was done at INFO level, (as per the decorator parameter). Additionally, the parameters that were provided to the function at various levels, are being logged accurately with respect to the correct contextual scope. In this example all arguments are named, which explains why `args=()`.

So, in principal, this decorator provides us a low-effort, unintrusive way to log entry and exit to the function. It also doesn't detract from any ad-hoc direct logging done within the function, but ensures that those logs are enriched with the context information from the surrounding function.

Let's deploy the function app using this, and observe some logs from the system.

#### Deploy the Function App

![Pipeline Dashboard](/assets/images/posts/structlog_kusto/Screenshot_ADO_PipelineDashboard.png){:class="img-fluid"}

The deployment pipeline includes the app service configuration, along with a build and deployment pipeline for the function app code. These are deployed together using a preconfigured service connection, backed by a service principal with owner-level access over the resource group pre-deployed into australia east.

Once deployed let's execute the following query to confirm diagnostic logs are being sent to a Log Analytics Workspace, and that we can see some logs.

```SQL
FunctionAppLogs
| where Category == 'Host.Function.Console'
| extend msg=replace(@"'", @'"', Message)
| extend msg_json=parse_json(msg)
| evaluate bag_unpack(msg_json)
```

It should be noted that even though JSON messages produced by structlog are well formatted JSON, they arrive at Log Analytics with single quotes (`'`) rather than double quotes (`"`). We execute a text replace function within Kusto to convert the `Message` column to valid JSON. 

We then parse the well formatted JSON string into a dynamic object, then use the `bag_unpack` operation to convert all the JSON keys into columns.

![Azure Log Analytics - Logs](/assets/images/posts/structlog_kusto/Screenshot_LAW_Logs.png){:class="img-fluid"}

Let's unpack JSON `msg` field we parsed out of the log message received by Log Analytics:

```json
{
    "event": "Exiting", 
    "client-ip": "10.0.32.17:59909", 
    "_Context__func_dir": "/home/site/wwwroot/Function", 
    "content-type": "application/x-www-form-urlencoded", 
    "x-arr-log-id": "19bf708e-5ca5-48a6-b29a-3259e965f346", 
    "x-forwarded-proto": "https", 
    "content-length": "62", 
    "x-appservice-proto": "https", 
    "args": [
        "Handler(secret_client=<azure.keyvault.secrets._client.SecretClient object at 0x7f1972cf0d30>)"
    ], 
    "accept": "*/*", 
    "was-default-hostname": "slg-fn.azurewebsites.net", 
    "_Context__func_name": "Function", 
    "x-arr-ssl": "2048|256|CN=Microsoft Azure TLS Issuing CA 01, O=Microsoft Corporation, C=US|CN=*.azurewebsites.net, O=Microsoft Corporation, L=Redmond, S=WA, C=US", 
    "host": "slg-fn.azurewebsites.net", 
    "x-site-deployment-id": "slg-fn", 
    "max-forwards": "9", 
    "_Context__trace_context": "<azure_functions_worker.bindings.tracecontext.TraceContext object at 0x7f198ba00e20>", 
    "disguised-host": "slg-fn.azurewebsites.net", 
    "x-forwarded-tlsversion": "1.2", 
    "x-original-url": "/api/Function?code=xxxxxxx", 
    "_Context__retry_context": "<azure_functions_worker.bindings.retrycontext.RetryContext object at 0x7f198ba00cd0>", 
    "appid": "a0656ce0a8e1fc8f5986cfe73767bdba", 
    "_Context__thread_local_storage": "<_thread._local object at 0x7f198b817f90>", 
    "function": "city_request_to_weather", 
    "_Context__invocation_id": "3c12f1fa-c1b8-47f7-b519-e7169b6500c7", 
    "city": "xxxxxxxx", 
    "state": "xxx", 
    "x-forwarded-for": "120.156.154.167:49292", 
    "x-waws-unencoded-url": "/api/Function?code=xxxxxxx", 
    "connection": "keep-alive", 
    "country": "xx", 
    "user-agent": "curl/8.1.2", 
    "level": "debug", 
    "logger": "function", 
    "timestamp": "2023-09-04T03:48:00.284861Z"
}

```

#### Metrics from Logs

Now that we have logs in Log Analytics, let's try to create a p99 latency metric from it.

The query below leverages the Kusto language to provide latency (in millis) for the call to the `city_request_to_weather` function, over 70% of the retained logs (70th percentile)

```sql
FunctionAppLogs
| where Category == 'Host.Function.Console'
| extend msg=replace(@"'", @'"', Message)
| extend msg_json=parse_json(msg)
| extend
    timestamp=todatetime(msg_json.timestamp),
    invocation_id=tostring(msg_json._Context__invocation_id),
    event=msg_json.event,
    scope=msg_json.scope,
    function=msg_json.function
| sort by timestamp asc
| project timestamp, invocation_id, event, scope, function
| where event in ("Exiting", "Entering") and function == 'city_request_to_weather'
| extend diff = case(event == "Exiting", datetime_diff('millisecond', timestamp, prev(timestamp)), 0)
| summarize latency = max(diff) by invocation_id
| summarize p99_latency = percentile(todouble(latency), 70)
```




### Key Recommendations for fun and profit

* Define a logging standard or guideline, compatible with structured logging
* Push structurally common log context-keys into framework code to guarantee consistency
* Bind context variables to narrow the programmatic logging interface and limit diagnostic information propagation
* Define your own logger detached from the root logger
* JSON is a compelling logging output format
* ERROR level only, starting with INFO for handoff-level only, then moving to metric based confirmation of handoff
* Add context dimensions/tags to metrics for high cardinality