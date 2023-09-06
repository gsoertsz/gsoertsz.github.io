---
layout: post
title: "Structlog in Azure for fun and profit"
categories: [python, structlog, azure, log_analytics, kusto]
kramdown:
    syntax_highlighter: coderay
---

*Structured logging in Azure with `structlog`, maximizes the analytic value in logs.*

<!--excerpt-above-->

For modern cloud systems, logging is an essential component of SIEM and the observability posture needed by DevOps teams to deal with high value systems of scale.

While structured logging, along with a plethora of log aggregation tools, has helped DevOps teams deal with the scale and heterogeneity of logs with a "single pane of glass", some fundamental complaints still exist around logging architectures. 

It's uncontroversial to assume developers just get it when it comes to logging. We've been inserting `print()` messages since forever, and logging code is added arbitrarily throughout the code as needed. Log levels are key to managing the granularity of log messages being emitted, and if there is certainty in the environments that populate the path to production, and we have a well written and relevant logging standard, the how and what of logging is a solved problem.

However, logging places a **downward pressure on software quality**. In order to log meaningful messages (even if they aren't pretty), throughout the code, developers need to construct distinct and descriptive event strings, rich with context and often composed of variables evaluated at the last minute. There isn't a general intuition about how to design a component for logging, and for the most part logging API's are globally configured and referenced accordingly. **Context smear**, and **log littering**, and embarrasingly, runtime log message failures, all comprise a potent, emergent characteristic of introducing logging to a meaningful production app. You wouldn't typically mock a logging interface, so logging API's escape the typical strategies we employ to manage software quality. It's not uncommon to mark lines containing log API calls as exempt from linting and code-coverage. Given a logging standard, and a halfway decent linting tool, it is possible to get style feedback for the use of logging calls, but again if you have a clear and opinionated logging standard, and a strong peer-review culture, there are diminising returns in codifying this. Generally, when you consider that over time, across services, and teams, log messages will be structured differently, emitted to varying degrees, at various levels, you get a sense of the heterogeniety and variety we will inevitably encounter when seeing all the logs in the log aggregation tool. This lack of consistency makes it difficult to yield analytic value from logs, not to mention the high coupling, and **query sprawl** between the source systems and the log analytics tool.

Costs are a huge concern, and 3rd party log aggregation tools will charge for ingestion, storage and retrieval, in addition to monthly subscription fees. As businesses themselves, they are more than willing to pass on cost of goods sold (COGS), and given than they are required to maintain a certain SLA for their customers - quite rightly - they also need to throttle your ingestion to protect other customers. Fundamentally, logs are data, and log aggregation tools are derivative data warehouses.

Over time, data warehouses suffer entropy if they don't have strong hygiene provisions (CI/CD, API and IaC, etc), or lend themselves to being organised naturally. Analytic artefacts, such as queries, notebooks and visualizations, alert configurations and dashboards, predominantly created via click-ops, proliferate wildly within the tool, until eventually they end up resembling one's kitchen pantry. When it comes to performing log analytics, the tool should facilitate **rapid, governed access**, and sustainable **short time to insight**, but log aggregation tools in general would benefit from broader governance, modelling and management practices that exist in modern data warehousing. Indeed, modern data warehouses are geared to maximizing the analytic value in data, so it follows that modern data management practices would be useful in log analytics. Taking this further, I'll leave as an exercise for the reader to evaluate the degree of convergence we'll see between modern data warehouses and log analytics tools specifically. 

Generally:

---

*A good logging architecture, regardless of scale, is one that seeks to maximize the sustained **analytic and diagnostic value in logs**, while minimizing the impact to **software quality**, and **performance**.*

---

We focus on the analytic and diagnostic value in logs in the above statement to offset the costs associated with centralised aggregated logging. Maximising this value allows logs to be involved in the virtuous growth cycle associated with using data generated from users to improve products. We also don't overly favour the analytic value to the detriment of software maintainability and performance, two key metrics in providing great products. 

To demonstrate some of these considerations, let's examine an Azure Function App written in python, that uses structlog to do some logging, along with a Log Analytics Workspace to explore the analytic value we can extract from logs.

#### Structlog - a good logging API

[Structlog](https://www.structlog.org/en/stable/) is a structured logging library for python projects. It has a run-of-the-mill logging API (log(...), debug(..), etc) and handler/formatter framework, but also provides good support for single threaded, as well as multi-threaded (flask) apps. No different from other logging API's, log levels are the typical way in which we manage app performance tradeoffs with logging.

Structlog also provides good support for [managing context information](https://www.structlog.org/en/stable/contextvars.html), such as `bind_contextvars(..)` which allows dictionary-like structures to be provided and bound to the logging context as key value pairs. These methods can be used to store key-value context variables and can prevent the need to propagate that detail throughout the code.

#### Standardization of logs with a logging python decorator

How might we achieve greater consistency in the log messages we generate throughout the code, and across apps?

By way of example, we can leverage structlog's context management functions, its general `log` message, and python decorators, to consistently log the entry to and exit from different functions within the implementation.

Leveraging decorators, allows us to simply mark functions that we are interested in. Once marked with the logging decorator, we can emit "Entering", and "Exiting" events prior to, and after execution respectively, as well as adding function details (name, args, etc) to the context information around logs. Additionally:

- If we do any direct logging within the marked function, we want those logs to come through with the same context information. 
- Once we exit the function we want the additional context (for the nested function) to be removed, and the prior context to be restored for the outer scope (if applicable)
- We want the logger used in the decorator to be customizable with respect to level to avoid surprises, and to control the overall log level centrally.

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

For the above functions, let's examine the log messages received

```terminal
2023-09-04 15:09:00 [debug] Entering                       args=() function=logging_context_helper_fn1 str_arg=some string
2023-09-04 15:09:00 [info ] This is the first fn call      args=() function=logging_context_helper_fn1 str_arg=some string
2023-09-04 15:09:00 [info ] Entering                       args=() function=logging_context_helper_fn2 str_arg=some string, with some more data
2023-09-04 15:09:00 [info ] This is the second function call args=() function=logging_context_helper_fn2 str_arg=some string, with some more data
2023-09-04 15:09:00 [info ] Exiting                        args=() function=logging_context_helper_fn2 str_arg=some string, with some more data
2023-09-04 15:09:00 [info ] After the nested call in the first scope args=() function=logging_context_helper_fn1 str_arg=some string
2023-09-04 15:09:00 [debug] Exiting                        args=() function=logging_context_helper_fn1 str_arg=some string
```

In the above scenario, entry/exit logging for `logging_context_helper_fn1` was done at debug level (default), whereas for `logging_context_helper_fn2`, entry/exit logging was done at INFO level, (as per the decorator parameter). Additionally, the parameters that were provided to the function at various levels, are being logged accurately with respect to the correct contextual scope.

So, in principal:

---

*The decorator approach helps us achieve **consistency** in log messages, and provides us a **low-effort**, and **unintrusive** way to log entry and exit to the function. It also enhances the **reliability** and **contextual richness** of ad-hoc direct logging done within the decorated function, by inheriting and managing context correctly, and **preventing the need to explicitly pass context** into the function or compute it at event message construction time.* 

---

#### Python Function App

Let's see this logging approach with structlog and decorators in action in a Python Function App integrated with Azure Log Analytics. All of the code that follows is available [on Github](https://github.com/gsoertsz/azure-function-app-structlog-example)

The function app uses the [Open Weather API](https://openweathermap.org/). Function Apps are a good example of at-scale cloud logs - we would very rarely log into the 'host' of a Function App (although in Azure this is possible)

The Open Weather API is a rich API service, that provides current weather information given a geocoded location (lat, long). Obviously this is a little inconvenient, as it requires that a plain english location (city, state, country etc) needs to be geocoded into coordinates, before we can derive weather information. Conveniently, the Open Weather API provides a geo-coding service as well. The function app, accepts a request containing a city, state, and country, then:

1. Uses the geocoding API to obtain the latitude (lat) and longitude (long) of the the location
2. Uses the weather API to retrieve the current weather for the lat/long acquired in (1)
3. Returns the current weather payload to the user

Because the Weather/Geocoding API requires an API key, the function app is deployed with an Azure KeyVault, and is configured with the secret to read. To use the Azure Key Vault, the Key Vault is configured to enable RBAC authorization, and the System Assigned managed identity for the function app is assigned the `Key Vault Secrets User` role on the Key Vault. This can be seen in the infrastructure template located [here](https://github.com/gsoertsz/azure-function-app-structlog-example/blob/03141827f905f92dc25030a3d746f4b895a30fc4/bicep-templates/functionapp.bicep#L166)

When servicing the request the function app retrieves the Weather API Key, then does the work above to provide the current weather back to the user. Below is an example request to the function app. The response we see below is the function app forwarding back on to the client, the response from the Open Weather API service.

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

As you can see from the above code, the code uses an Azure Managed Identity to access the Key Vault to retrieve the API Key for interacting with the Open Weather Service. 

From a logging point of view the `loggingutils.log_context(...)` decorator is applied to the `handle(...)` function and the `city_request_to_weather` function. For each of these the globally configured logger is passed in and the default log level (DEBUG) is not overridden.

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

- Prints stack traces for errors
- Format's timestamps as ISO-8601
- Prints logger name and level
- Prints the log message in JSON format

This setup will provide a pretty useable structured logging configuration.

Let's deploy the function app using this, and observe some logs from the system.

#### Deploy the Function App

![Pipeline Dashboard](/assets/images/posts/structlog_kusto/Screenshot_ADO_PipelineDashboard.png){:class="img-fluid"}

The deployment pipeline includes the app service configuration, along with a build and deployment pipeline for the function app code. These are deployed together using a preconfigured service connection, backed by a service principal with owner-level access over the resource group pre-deployed into australia east.

Once deployed let's execute the following query to confirm diagnostic logs are being sent to a Log Analytics Workspace, and that we can see some logs.

#### Azure Log Analytics

With the function app configured to send all logs to a Log Analytics Workspace, we are able to query the logs 

```SQL
FunctionAppLogs
| where Category == 'Host.Function.Console'
| extend msg=replace(@"'", @'"', Message)
| extend msg_json=parse_json(msg)
| evaluate bag_unpack(msg_json)
```

It should be noted that even though JSON messages produced by structlog are well formatted JSON, they arrive at Log Analytics with single quotes (`'`) rather than double quotes (`"`). We execute a text replace function within Kusto to convert the `Message` column to valid JSON. 

We then parse the well formatted JSON string into a dynamic object, then we can use the `bag_unpack` operation to convert all the JSON keys into columns.

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

Now that we have logs in Log Analytics, let get a measure of latency.

The query below leverages the Kusto language to provide latency (in millis) for the call to the `city_request_to_weather` function across all the requests.

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
| summarize avg_latency = avg(latency)
```

Let's first see if we can calculate the latency experienced on a per-request basis. For this we can extract and use the `invocation_id`.

![Latencies Summarised by Invocation Id](/assets/images/posts/structlog_kusto/Screenshot_LAW_SummarizeByInvocation_Id.png){:class="img-fluid"}

Then we can use kusto to get the overall average latency across all the requests.

![Overall Average Latency](/assets/images/posts/structlog_kusto/Screenshot_LAW_AverageLatency.png){:class="img-fluid"}

#### Alerts from Logs

Let's alert if the latency is above a certain threshold. By configuration, within Log Analytics, its possible to define an alert based on log metrics. Below is an extract from the bicep template, containing the configuration. 

```json
...
resource logQueryWeatherLatencyAlert 'Microsoft.Insights/scheduledQueryRules@2023-03-15-preview' = {
  name: logQueryAlertName
  location: location
  properties: {
    description: 'The Weather API latency exceeds acceptable thresholds'
    severity: 2
    enabled: true
    actions: {
      actionGroups: []
    }
    scopes: [
      logAnalyticsWorkspace.id
    ]
    evaluationFrequency: 'PT5M'
    windowSize: 'PT5M'
    criteria: {
      allOf: [
        {
          query: 'FunctionAppLogs | where Category == \'Host.Function.Console\' | extend msg=replace(@"\'", @\'"\', Message) | extend msg_json=parse_json(msg) | extend timestamp=todatetime(msg_json.timestamp), invocation_id=tostring(msg_json._Context__invocation_id), event=msg_json.event, scope=msg_json.scope, function=msg_json.function | sort by timestamp asc | project timestamp, invocation_id, event, scope, function | where event in ("Exiting", "Entering") and function == \'city_request_to_weather\' | extend diff = case(event == "Exiting", datetime_diff(\'millisecond\', timestamp, prev(timestamp)), 0) | summarize latency = max(diff) by invocation_id | summarize avg_latency = avg(latency)'
          metricMeasureColumn: 'avg_latency'
          operator: 'GreaterThan'
          timeAggregation: 'Average'
          threshold: 500
          failingPeriods: {
            numberOfEvaluationPeriods: 1
            minFailingPeriodsToAlert: 1
          }
        }
      ]
    }
    autoMitigate: true
  }
}
...
```

Let's have a look at the alert in the UI.

![Alert Configuration](/assets/images/posts/structlog_kusto/Screenshot_LAW_AlertConfiguration.png){:class="img-fluid"}

Details of the alert configuration are below.

![Alert Configuration Detail](/assets/images/posts/structlog_kusto/Screenshot_LAW_AlertConfigurationDetails.png){:class="img-fluid"}

If we submit a request to the function app a few times, and wait for the alert to be evaluated, we can observe that the alert get's raised. Once the average falls below the threshold it get's auto-resolved.

![Alert Timeline](/assets/images/posts/structlog_kusto/Screenshot_LAW_AlertTimeline.png){:class="img-fluid"}

This type of query would not be possible unless we had structure, context and consistency in our log messages, and with these we can already see that we can extract more from logs than just diagnostics or a log trace.

#### Azure's Analytic Ecosystem

It's possible to forward logs to a number of destinations. 

![Function App Diagnostic Settings](/assets/images/posts/structlog_kusto/Screenshot_FunctionApp_DiagnosticSettingOptions.png){:class="img-fluid"}

It is also possible to export logs to a storage account from Log Analytics, via the `Data Export` functionality.

![Data Export](/assets/images/posts/structlog_kusto/Screenshot_LAW_DataExport.png){:class="img-fluid"}

Once in a storage account, the log data can be ingested into Azure Data Explorer, Azure Synapse etc. Also, via a streaming spark application in Azure Synapse its possible to continuously ingest data from an eventhub destination into a Synapse SQL table for analysis using a dedicated or serverless SQL pool, leveraging full Synapse SQL, rather than just Kusto. How to do this is outside the scope of this blog.

While there will be costs associated with this integration, and storage costs both for the storage account and the destination, the number of hops needed, and the complexity of the integration with sophisticated analytics tools is compelling. Furthermore, if Azure already hosts your data warehouse in some of these technologies, logs are at your doorstep.

Overall:

---

*In Azure, logs are easily integrated into a cloud-native capable, general purpose analytic tool. If these tools are already part of your data warehouse eco-system, the analytic value in logs is maximized and logs contribute to a virtuous growth cycle.*

---

### Key Takeaways

I found the Azure, Python, structlog and Log Analytics experience really cool. 

Logs are data, a form of authentic, ground truth telemetry, that should be treated as value-carrying business events, but data quality in logs is chronically bad. Log analytic tools are necessary, but expensive, and beyond diagnostics, we must extract analytic value to offset the costs, allowing logs to contribute to the virtuous growth cycle

With `structlog` and python decorators we are able to improve the consistency of our log messages, while also addressing inherent context-smear and log littering, without diminishing our ability to manage the performance impact of fine grained logs, or developer discretion in relation to where to insert log messages.

Well structured, high-dimensional, consistent, structured logs, in turn, maximizes the extraction of both diagnostic and analytic value in logs via Azure Log Analytics with KQL, while also preserving the single pane of glass, a key pillar of observability at scale.

With Azure's analytic and event-based messaging eco-system, logs can be easily incorporated into a centralized data warehouse to further exploit data-centric governance, modelling and analytic capabilities that are already optimized to maximise the hygiene and analytic value of data.





