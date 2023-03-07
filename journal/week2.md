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
# CloudWatch logs ------------------
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

## Homework Challenges
