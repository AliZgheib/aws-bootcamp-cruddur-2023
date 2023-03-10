# Week 2 — Distributed Tracing

## Required Homeworks/Tasks

### CloudWatch Logs

#### Setup CloudWatch with our Flask app

1. add ```watchtower``` to the ```requirements.txt```

2. add the necessary imports 

```
import watchtower
import logging
from time import strftime
```

3. setup watchtower to ouput logs to **cruddur-backend-flask** log group on AWS CloudWatch

```
LOGGER = logging.getLogger(__name__)
LOGGER.setLevel(logging.DEBUG)
console_handler = logging.StreamHandler()
cw_handler = watchtower.CloudWatchLogHandler(log_group='cruddur-backend-flask')
LOGGER.addHandler(console_handler)
LOGGER.addHandler(cw_handler)
```

4. configure the **LOGGER** to ouput request info after each API call

```
@app.after_request
def after_request(response):
    timestamp = strftime('[%Y-%b-%d %H:%M]')
    LOGGER.info('%s %s %s %s %s %s', timestamp, request.remote_addr, request.method, request.scheme, request.full_path, response.status)
    return response
```

#### Verify the integration on CloudWatch

1. we run our application using ```docker compose up``` command and we open the home page

2. we verify that **cruddur-backend-flask** was created successfully on AWS CloudWatch log groups

![Cloud watch log group](assets/week2/cloudwatch-logs-1.PNG)

3. we verify that **watchtower** is outputting logs on our log group

![Cloud watch log event](assets/week2/cloudwatch-logs-2.PNG)

### Rollbar

#### Setup Rollbar project

1. we visit [rollbar](https://rollbar.com/) website

2. we create a new project **cruddur-backend-flask**

#### Setup Rollbar with our Flask app

1. add ```blinker``` and ```rollbar``` to ```requirements.txt```

2. We need to set our access token

```
export ROLLBAR_ACCESS_TOKEN=""
gp env ROLLBAR_ACCESS_TOKEN=""
```

3. Add to backend-flask for ```docker-compose.yml```

```
ROLLBAR_ACCESS_TOKEN: "${ROLLBAR_ACCESS_TOKEN}"
```

4. add the necessary imports

```
import rollbar
import rollbar.contrib.flask
from flask import got_request_exception
```

5. setup our app to ouput logs to **cruddur-backend-flask** on Rollbar

```
rollbar_access_token = os.getenv('ROLLBAR_ACCESS_TOKEN')
@app.before_first_request
def init_rollbar():
    """init rollbar module"""
    rollbar.init(
        # access token
        rollbar_access_token,
        # environment name
        'production',
        # server root directory, makes tracebacks prettier
        root=os.path.dirname(os.path.realpath(__file__)),
        # flask already sets up logging
        allow_logging_basic_config=False)

    # send exceptions from `app` to rollbar, using flask's signal system.
    got_request_exception.connect(rollbar.contrib.flask.report_exception, app)
```

6. We add a new endpoint in ```app.py``` to test our rollbar integration

```
@app.route('/rollbar/test')
def rollbar_test():
    rollbar.report_message('Hello World!', 'warning')
    return "Hello World!"
```

#### Verify the integration on Rollbar

1. we run our application using ```docker compose up``` command

2. we verify that the ```/rollbar/test``` endpoint is returning a 200 response

![Rollbar endpoint response](assets/week2/rollbar-1.PNG)

3. we verify that we are receiving logs on Rollbar

![Rollbar logs](assets/week2/rollbar-2.PNG)

### HoneyComb 

#### Introduction

Instead of doing the setup that was done live. I'll be implementing our backend flask application to write to **otel collector**. this is more scalable approach and will allow us to add other backend services and our react application as well in the future.

More information can be found on the official [OpenTelemetry](https://opentelemetry.io/docs/collector/) website

Here's a simple illustration to showcase the approach that we are trying to implement for our cruddur application.

![OpenTelemetry and Cruddur](assets/week2//opentelemetry-collector.jpg)

#### Setup HoneyComb project

1. we visit [HoneyComb](https://www.honeycomb.io/) website

2. Created a new enviroment and retrieve our api key

#### Setup HoneyComb with our Flask app

1. add environment variables

```
export HONEYCOMB_API_KEY=""
gp env HONEYCOMB_API_KEY=""
```

2. add the list below to the ```requirements.txt```

```
opentelemetry-api 
opentelemetry-sdk 
opentelemetry-exporter-otlp-proto-http 
opentelemetry-instrumentation-flask 
opentelemetry-instrumentation-requests
```

3. add the following environment variables to our **backend-flask** service.

```
version: "3.8"
services:
    backend-flask:
        environment:

            # other environment variables

            OTEL_EXPORTER_OTLP_ENDPOINT: "http://otel-collector:4318"
            OTEL_SERVICE_NAME: "cruddur-backend-flask"

    # other services
```

this will tell our backend flask application to send the backend traces to the **otel-collector** that we are going to create shorly.

4. add the **otel-collector** service to our ```docker-compose.yml``` file

```
version: "3.8"
services:

  # other services

  otel-collector:
    image: otel/opentelemetry-collector:0.67.0
    environment:
      HONEYCOMB_API_KEY: "${HONEYCOMB_API_KEY}"
      FRONTEND_URL_HTTPS: "https://3000-${GITPOD_WORKSPACE_ID}.${GITPOD_WORKSPACE_CLUSTER_HOST}"
      FRONTEND_URL_HTTP: "http://3000-${GITPOD_WORKSPACE_ID}.${GITPOD_WORKSPACE_CLUSTER_HOST}"
    command: ["--config=/etc/otel-collector-config.yaml"]
    volumes:
      - ./otel-collector-config.yaml:/etc/otel-collector-config.yaml
    ports:
      - "1888:1888"   # pprof extension
      - "8888:8888"   # Prometheus metrics exposed by the collector
      - "8889:8889"   # Prometheus exporter metrics
      - "13133:13133" # health_check extension
      - "4317:4317"   # OTLP gRPC receiver
      - "4318:4318"   # OTLP http receiver
      - "55679:55679" # zpages extension
```
5. create the ```otel-collector-config.yaml``` config file that will be used by the **otel collector**

```
receivers:
  otlp:
    protocols:
      http:
        cors:
          allowed_origins:
            - "${FRONTEND_URL_HTTPS}" # this will allow trace requests from the frontend ( To solve CORS issues )
            - "${FRONTEND_URL_HTTP}"  # It's not required for the backend service
processors:
  batch:

exporters:
  otlp:
    endpoint: "api.honeycomb.io:443"
    headers:
      "x-honeycomb-team": "${HONEYCOMB_API_KEY}"

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlp]
```

5. add necessary changes to ```app.py```

```
from opentelemetry import trace
from opentelemetry.instrumentation.flask import FlaskInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
```

```
# Initialize tracing and an exporter that can send data to Honeycomb
provider = TracerProvider()
processor = BatchSpanProcessor(OTLPSpanExporter())
provider.add_span_processor(processor)
trace.set_tracer_provider(provider)
tracer = trace.get_tracer(__name__)
```

```
# Initialize automatic instrumentation with Flask
app = Flask(__name__)
FlaskInstrumentor().instrument_app(app)
RequestsInstrumentor().instrument()
```
#### Validate the automated backend instrumentation

1. we run our application using ```docker compose up``` command

2. we visit our cruddur application home page.

3. we validate on HoneyComb dashboard that we have a new dataset **cruddur-backend-flask** created and that we are successfully receiving data from our **backend-flask** service
![Cruddur Backend flask](assets/week2/honeycomb-1.PNG)

### X-Ray

#### Setup X-Ray with our Flask app

1. add ```aws-xray-sdk``` to the ```requirements.txt``` file

2. add the following environment variables to our **backend-flask** service.

```
version: "3.8"
services:
    backend-flask:
        environment:

            # other environment variables

            AWS_XRAY_DAEMON_ADDRESS: "xray-daemon:2000"

    # other services
```

this will tell our backend flask application to send the backend traces to the **xray-daemon** that we are going to create shorly.

3. add the **xray-daemon** service to our ```docker-compose.yml``` file

```
version: "3.8"
services:

  # other services

  xray-daemon:
      image: "amazon/aws-xray-daemon"
      environment:
          AWS_ACCESS_KEY_ID: "${AWS_ACCESS_KEY_ID}"
          AWS_SECRET_ACCESS_KEY: "${AWS_SECRET_ACCESS_KEY}"
          AWS_REGION: "us-east-1"
      command:
          - "xray -o -b xray-daemon:2000"
      ports:
          - 2000:2000/udp
```
4. create a group on aws-xray to ensure we could group our traces

```
aws xray create-group \
--group-name "cruddur-backend-flask" \
--filter-expression "service(\"cruddur-backend-flask\")"
```

5. we create a sampling as follow:

```
{
    "SamplingRule": {
        "RuleName": "cruddur-backend-flask",
        "ResourceARN": "*",
        "Priority": 9000,
        "FixedRate": 0.1,
        "ReservoirSize": 5,
        "ServiceName": "cruddur-backend-flask",
        "ServiceType": "*",
        "Host": "*",
        "HTTPMethod": "*",
        "URLPath": "*",
        "Version": 1
    }
}
```

```
aws xray create-sampling-rule --cli-input-json file://aws/json/xray.json
```

6. add necessary changes to ```app.py```

```
from aws_xray_sdk.core import xray_recorder
from aws_xray_sdk.ext.flask.middleware import XRayMiddleware
```

```
xray_recorder.configure(service='cruddur-backend-flask')
XRayMiddleware(app, xray_recorder)
```
#### Validate the automated backend instrumentation

1. we run our application using ```docker compose up``` command

2. we navigate around the pages of our Currdur application ( ex: home, notifications, messages, etc.. )

3. we validate on AWS X-Ray dashboard that we have new traces under our **cruddur-backend-flask** group

![AWS X-Ray](assets/week2/x-ray-1.PNG)
![AWS X-Ray](assets/week2/x-ray-2.PNG)

## Homework Challenges

### Add custom instrumentation to Honeycomb & run custom queries in Honeycomb

#### user_activities - popular users on Cruddur

we add custom instrumentation to the **user_activities** service and we set the **user_handle** as a custom attribute. this will allow us to discover the most popular and searched users on Cruddur platform.

1. we update the ```user_activities.py``` file

```
# other imports..

from opentelemetry import trace
tracer = trace.get_tracer(__name__)
```

```
# inside the class run function

span = trace.get_current_span()
span.set_attribute("app.user_handle", user_handle)
```

2. we try accessing different profiles on Cruddur ( ex /@andrewbrown or /@alizgheib ) and validate the results on HoneyComb

![HoneyComb](assets/week2/honeycomb-2.PNG)

#### show_activity - popular activities on Cruddur

we add custom instrumentation to the **show_activity** service and we set the **activity_uuid** as a custom attribute. this will allow us to discover the most popular activities on Cruddur platform.

1. we update the ```show_activity.py``` file

```
# other imports..

from opentelemetry import trace
tracer = trace.get_tracer(__name__)
```

```
# inside the class run function

span = trace.get_current_span()
span.set_attribute("app.activity_uuid", activity_uuid)
```

2. we try accessing different activities on Cruddur ( ex /activity-1 or /activity-2 ) and validate the results on HoneyComb

![HoneyComb](assets/week2/honeycomb-3.PNG)


#### search_activities - popular searche terms on Cruddur

we add custom instrumentation to the **search_activities** service and we set the **search_term** as a custom attribute. this will allow us to discover the most searched activities on Cruddur platform.

1. we update the ```search_activities.py``` file

```
# other imports..

from opentelemetry import trace
tracer = trace.get_tracer(__name__)
```

```
# inside the class run function

span = trace.get_current_span()
span.set_attribute("app.search_term", search_term)
```

2. we try searching different activities on Cruddur ( ex /search?term=AWS or /search?term=andrewbrown etc... ) and validate the results on HoneyComb

![HoneyComb](assets/week2/honeycomb-4.PNG)

### Instrument Honeycomb for the frontend-application

#### Introduction

HoneyComb provides a good [documentation](https://docs.honeycomb.io/getting-data-in/opentelemetry/browser-js/) on how we can intergrate its service on our frontend application and allow us to have a full visibility of the traces between the frontend and the backend.

there are multiple configurations that can allow us to get to the end results:

- Browser Code Configuration (Exposing Your Key)
- OpenTelemetry Collector Configuration
- Custom Proxy Configuration

we will go with the 2nd appraoch because its the most scalable one and it doesnt involve us exposing our API keys in the frontend application.

#### Setup HoneyComb with our React app

1. To instrument your Web page, add the following packages:

```
npm install --save \
    @opentelemetry/api \
    @opentelemetry/sdk-trace-web \
    @opentelemetry/exporter-trace-otlp-http \
    @opentelemetry/context-zone
```

2. The OpenTelemetry initialization needs to happen as early as possible in the webpage. Accomplish this by creating an initialization file, similar to the JavaScript example below.

```js
// tracing.js
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';
import { WebTracerProvider, BatchSpanProcessor } from '@opentelemetry/sdk-trace-web';
import { ZoneContextManager } from '@opentelemetry/context-zone';
import { Resource }  from '@opentelemetry/resources';
import { SemanticResourceAttributes } from '@opentelemetry/semantic-conventions';

const exporter = new OTLPTraceExporter({
  url: `${process.env.REACT_APP_OTEL_EXPORTER_OTLP_ENDPOINT}/v1/traces`,
});

const provider = new WebTracerProvider({
  resource: new Resource({
    [SemanticResourceAttributes.SERVICE_NAME]:
      process.env.REACT_APP_OTEL_SERVICE_NAME,
  }),
});
provider.addSpanProcessor(new BatchSpanProcessor(exporter));
provider.register({
  contextManager: new ZoneContextManager(),
});
```

3. Then, load the initialization file at the top of your web page’s header or entry point file.

```js
// index.js
import './tracing.js'

// ...rest of the app's entry point code
```

4. Connecting the Frontend and Backend Traces

It is possible to connect your frontend request traces to your backend traces, which allows you to trace a request all the way from your browser through your distributed system. To connect your frontend traces to your backend traces, you need to include the trace context header in the request. This can be done as follow:

```
npm install --save \
    @opentelemetry/instrumentation \
    @opentelemetry/instrumentation-xml-http-request \
    @opentelemetry/instrumentation-fetch
```

```js
registerInstrumentations({
  instrumentations: [
    new XMLHttpRequestInstrumentation({
      propagateTraceHeaderCorsUrls: [
        new RegExp(`${process.env.REACT_APP_BACKEND_URL}`, "g"),
      ],
    }),
    new FetchInstrumentation({
      propagateTraceHeaderCorsUrls: [
        new RegExp(`${process.env.REACT_APP_BACKEND_URL}`, "g"),
      ],
    }),
  ],
});
```

5. expose **otel-collector** port to the public in the ```.gitpod.yml```

```
ports:

  # other ports

  - name: otel-collector
    port: 4318
    visibility: public
```

6. update ```app.py``` CORS configs:

old version:
```
  allow_headers="content-type,if-modified-since",
```
new version:
```
  allow_headers=["content-type", "if-modified-since", "traceparent"],
```

#### Validate the automated frontend instrumentation

1. we run our application using ```docker compose up``` command

2. we visit our cruddur application home page.

3. we validate on HoneyComb dashboard the new trace(s)

![Cruddur Backend flask](assets/week2/honeycomb-5.PNG)


### Add custom instrumentation to AWS X-Ray

#### Setup custom instrumentation to the notifications service

1. add the neccessary imports

```
from aws_xray_sdk.core import xray_recorder
```

2. start a subsegment 

```
subsegment = xray_recorder.begin_subsegment('notifications_activities_subsegment')
```

3. add annotation to the subsegment

```
subsegment.put_annotation('results_length', len(results))
subsegment.put_annotation('request_time', now.isoformat())
subsegment.put_annotation('request_domain', request.url)
subsegment.put_annotation('request_path', request.path)
subsegment.put_annotation('request_method', request.method)
```

4. close the subsegment

```
xray_recorder.end_subsegment()
```

#### Validate the custom instrumentation on AWS X-Ray

1. we run our application using ```docker compose up``` command

2. we visit our cruddur application **notifications page**.

3. we validate the subsegment and annotations on AWS X-Ray

![Cruddur Backend flask](assets/week2/x-ray-3.PNG)