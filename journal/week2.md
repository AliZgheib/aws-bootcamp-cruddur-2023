# Week 2 — Distributed Tracing

## Required Homeworks/Tasks

### CloudWatch Logs

#### Install the required libraries

1. add ```watchtower``` to the ```requirements.txt```

2. install the libraries/dependencies

```
pip install -r requirements.txt
```

#### Setup CloudWatch with our Flask app

1. add the necessary imports 

```
import watchtower
import logging
from time import strftime
```

2. setup watchtower to ouput logs to **cruddur-backend-flask** log group on AWS CloudWatch

```
LOGGER = logging.getLogger(__name__)
LOGGER.setLevel(logging.DEBUG)
console_handler = logging.StreamHandler()
cw_handler = watchtower.CloudWatchLogHandler(log_group='cruddur-backend-flask')
LOGGER.addHandler(console_handler)
LOGGER.addHandler(cw_handler)
```

3. configure the **LOGGER** to ouput request info after each API call

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

#### Install the required libraries

1. add ```blinker``` and ```rollbar``` to ```requirements.txt```

2. install the libraries/dependencies

```
pip install -r requirements.txt
```

#### Setup Rollbar with our Flask app

1. We need to set our access token

```
export ROLLBAR_ACCESS_TOKEN=""
gp env ROLLBAR_ACCESS_TOKEN=""
```

2. Add to backend-flask for ```docker-compose.yml```

```
ROLLBAR_ACCESS_TOKEN: "${ROLLBAR_ACCESS_TOKEN}"
```

3. add the necessary imports

```
import rollbar
import rollbar.contrib.flask
from flask import got_request_exception
```

4. setup our app to ouput logs to **cruddur-backend-flask** on Rollbar

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

5. We add a new endpoint in ```app.py``` to test our rollbar integration

@app.route('/rollbar/test')
def rollbar_test():
    rollbar.report_message('Hello World!', 'warning')
    return "Hello World!"

#### Verify the integration on Rollbar

1. we run our application using ```docker compose up``` command

2. we verify that the ```/rollbar/test``` endpoint is returning a 200 response

![Rollbar endpoint response](assets/week2/rollbar-1.PNG)

3. we verify that **watchtower** is outputting logs on our log group

![Rollbar logs](assets/week2/rollbar-2.PNG)

## Homework Challenges
