# [fit] Serverless Architecture
# [fit] for
# [fit] Powerful Data Pipelines
<br>

## Jason A Myers
![inline, 100%](juice.png)

^ Welcome to Leveraging Serverless Architecture for Powerful Data Pipelines. My start with data pipelines was as an aspiring analytical chemist intern while in high school.

---

![fit,inline](c3880.jpg)

Credit: Horst Felske and Fritz Schiemann
[ex-convex.org](http://www.ex-convex.org/fritz/exConvex/sican/index.html)

^ I was working at an air force base using a convex mainframe like you see here to enter and calculate numbers from a real analytical chemist's experiments. That started this strange fascination with data or ETL pipelines

---

# Issues

- Scheduled
- Complex
- Scaling
- Recovery

^ Traditional pipelines are often kicked off at a scheduled time and depending on resources we may need to be careful about overlap. Pipelines with multiple phases and interdependencies quickly become too complex to reason about sanely. I know I've certainly loaded absolute trash data via ETL all because of not understanding some part of a pipeline. This complexity makes it difficult to scale pipelines as our data sets rapidly expand or evolve. And what happens when stuff goes south: retry, replay or Knuth forbid syncing ever need to be done.

---

![fit,inline](stressed.jpg)

Credit: Anonymous

---

# Python Toolkits and Services

- [Luigi](http://luigi.readthedocs.io/en/stable/index.html)
- [Airflow](https://airflow.incubator.apache.org/)
- [AWS Data Pipeline](https://aws.amazon.com/datapipeline/)

^ Thankfully now, we have wonderful toolkits and services like Luigi, Airflow, and Data Pipeline to let help us focus on the sources, transformations, outputs, and notifications in a sane manner, as well as, handling a bit of what happens when we don't execute properly with things like back fill, task retry etc. However, I'm often still needing some other component to help react to events (triggering), scale out (parallelization), or mix in streaming sources. I don't say these things to pick on these tools, as I don't think all of this is their responsibility and they've all made wise choices about what the tools are good at and what is left to the implementor. For me, I like to fill in some of these gaps with...

---

# [fit] SERVERLESS

^ SERVERLESS! That's right go ahead and fill in that square on your unicorn buzzword bingo card. There is so much hype around serverless that it can feel a bit too good to be true. I wanna set all that hype aside and focus on just one of the parts of a serverless architecture...

---

![fit](so-doge-much-hype-very-excite.jpg)


---

# Cloud Functions

^ Cloud Functions are wonderful little Event Driven functions that we can upload as just the function and it's associated requirements, without concern for what will execute it. I called them wonderful because you can truly not worry about how they are run until you have to... They feature Automatic STDOUT logging and Permission Based Execution makes working with them in a auditable and "prescribed" manner possible. And for the next wonderful part...

---
![fit](lambda-logo.png)
![fit](gcf-logo.png)
![fit](azurefunctions.png)

^ Many cloud providers offer them...  You can couple then with Google's wonderful Container services, Azure's beautiful ML APIs, and of course Amazon's wide array of data analysis tools. All three of them work quite well and have a slightly varying feature set but the overall concepts apply.

---

![fit](lambda-logo.png)

^ For this talk, We're going to focus on AWS Lambda since it's been around the longest. It supports both python and legacy python, node, .net and more

---

# Simple Pipeline overview

![fit, inline](simple-lambda-1.png)

^ Let's take a look at a simple pipeline. For me a lot of my data pipelines start with a file upload or a scheduled event. For this simpler overview, let's start with a file upload. A file is uploaded to an S3 bucket, which fires an AWS event that can have an array of targets...

---

# Simple Pipeline overview

![fit, inline](simple-lambda-2.png)

^ but let's send ours to one of those spiffy lambda cloud functions. SERVERLESS! When this function gets an event, it can decide which pipeline this file belongs to send it to that pipeline. Now it's possible to directly invoke the pipeline, but I like to use a queue or an event stream (more on this later) and let an event listen to that. It removes the direct coupling that might get in our way latter.

---
# Simple Pipeline overview

![fit, inline](simple-lambda-3.png)

^ For this example, we're going to use a simple SQS queue to hold each pipeline runs that need to occur.

---
# Simple Pipeline overview

![fit, inline](simple-lambda-4.png)

^ We'll have the SQS queue trigger another function when an item is is added into the queue. This function will grab the file from S3 and break it up into records to be written to our database.
---
# Simple Pipeline overview

![fit, inline](simple-lambda-5.png)

^ Each one of these records will be used to invoke yet another function that writes the data into a datastore of some sort. In our case, I'm using amazon redshift here.  If we needed to do additional processing on our data, or even just some part of the records
---
# Simple Pipeline overview

![fit, inline](simple-lambda-6.png)

^ We can add another function to perform that transformation and invoke it for each record we need to transform. Then the transform function can send it to the writer for safekeeping.

---
# Simple Pipeline overview

![fit, inline](simple-lambda-scaling.png)

^ The wonderful part of this setup is that it scales exceptionally well. Each S3 event creates an invocation of our s3 handler function, same for our queue, and same for our reader and transformer!  This is a simple example, and we're going to show a deeper example, but let's talk about these functions for a section. We're gonna start by talking about the tools available to us to create them

---

# Serverless Python Tools

- Zappa
- Apex
- Chalice
- Serverless

^ SERVERLESS! in python we have a wide array of choices to work with the excellent Zappa framework, Apex, Chalice, and the SERVERLESS! framework itself. I tend to use the top two the most depending on what I'm doing. If I'm building an web app, API or proxy I tend to use Zappa; however, for cloud functions I really like

---

# Apex

- Multiple Environment Support
- Function Deployment
- Infrastructure as code via Terraform

^ Apex. This isn't a slam to the other tools it's just a personal preference. I like the way Apex handle multiple environments by separating the configuration and infrastructure from the code. I like the way it performs the function deployment with simple commands that take environments as arguments, and finally i like that I can use terraform to build the infrastructure components I need to support my lambda functions all in one tool.

---
# Project Structure

```
├── functions
│   └── listener
│       └── main.py
├── infrastructure
│   ├── dev
│   │   ├── main.tf
│   │   ├── outputs.tf
│   │   └── variables.tf
│   ├── prod
│   │   ├── main.tf
│   │   ├── outputs.tf
│   │   └── variables.tf
├── project.json
└── project.prod.json
```

^ This is what a typically layout of an Apex project looks like. I have a collection of functions in a directory with a directory per function and it's associated code. Then I have infrastructure configuration for each environment in nested format, and finally the project configurations for each environment that direct apex on what to do.

---
# Apex package.json

```
{
  "name": "listener",
  "description": "S3 File Listener",
  "runtime": "python3.6",
  "memory": 128,
  "timeout": 5,
  "role": "arn:aws:iam::ACOUNTNUM:role/listen_lambda_function",
  "environment": {},
  "defaultEnvironment": "dev"
}
```

^ Here is an example a project configuration. It sets the details about the project, function configuration options, and environment details.

---

# S3 Event Handler

```
import logging

import boto3

log = logging.getLogger()
log.setLevel(logging.DEBUG)

def get_bucket_key(event);
    bucket = event['Records'][0]['s3']['bucket']['name']
    key = event['Records'][0]['s3']['object']['key']
    return bucket, key

def handle(event, context):
    log.info('{}-{}'.format(event, context))
    bucket_name, key_name = get_bucket_key(event)
```

^ Here is the first part of an example S3 function handler. The critical part is this handle method. This is the code that will get run when the lambda function is invoked. Every lambda function needs to accept an event and a context. The event contains the data of the event that triggered it, and the context contains any additional information supplied that might relate to the handling of the lambda function. In practice through, we're focused on what we are getting in the event. Here you can see we parse the event records list and grab the details of the file upload that triggered the event in our get_bucket_key function.

---

# S3 Event Handler (cont.)

```
    values = {
        'bucket_name': bucket_name,
        'key_name': key_name,
        'timestamp': datetime.utcnow().isoformat()
    }

    client = boto3.client('sqs')
    client.publish(
        TopicArn=topic_arn,
        Message=json.dumps(values)
    )
```

^ The we place the item on to an SQS queue so that it can be picked up later. Now this function is ready to go, but we need to connect it to a few services to make it SERVERLESS! With apex we can build and provide permission to these services using terraform! Terraform is a tool by Hashicorp that let's us define infrastructure as code. Let's look at an example of that. Our function will have an IAM role (think users but for nonhuman usage) and we want that role to be able to publish logging output to Cloudwatch amazon's nifty logging service.

---

# AWS Logging Permissions

```
data "aws_iam_policy_document" "listener_logging" {
  statement {
    sid       = "AllowRoleToOutputCloudWatchLogs"
    effect    = "Allow"
    actions   = ["logs:*"]
    resources = ["*"]
  }
}

resource "aws_iam_policy" "listener_logs" {
  name        = "listener_logs"
  description = "Allow listener to log operations"
  policy      = "${data.aws_iam_policy_document.listener_logging.json}"
}
```
^ We begin by build a policy document and creating a policy from it.

---

# AWS IAM Role Assumption

```
data "aws_iam_policy_document" "listener_lambda_assume_role" {
  statement {
    sid     = "AllowRoleToBeUsedbyLambda"
    effect  = "Allow"
    actions = ["sts:AssumeRole"]

    principals {
      type        = "Service"
      identifiers = ["lambda.amazonaws.com"]
    }
  }
}

resource "aws_iam_role" "listener_lambda_function" {
  name               = "listener_lambda_function"
  assume_role_policy = "${data.aws_iam_policy_document.listener_lambda_assume_role.json}"
}
```

^ Next we create the ability for our IAM role to be used by lambda with an AssumeRole Policy, and build the role with that policy associated.

---

# AWS Policy Attachment

```
resource "aws_iam_policy_attachment" "listener_logs_attach" {
  name       = "listener_logs_attach"
  roles      = ["${aws_iam_role.listener_lambda_function.name}"]
  policy_arn = "${aws_iam_policy.listener_logs.arn}"
}
```

^ Then finally, we attach the logging policy to the role as well. This was several steps, but creating this as code means that we can create and delete these AWS resource at will without clicking through the AWS console. It also means that I can replay this in several different environments so that my dev, staging and prod all work the same. In our case we use separate AWS accounts for each of our environments, but you can add the environment as a variable into your terraform code to make the roles for each environment have different names.

---

# Deploy Function and Infrastructure

```
# apex deploy
# apex infra plan
# apex infra apply
```

^ These three commands will deploy our cloud function, and set up our related Infrastructure. The plan allows us to see what infrastructure changes will be made before we make them with apply.

---

# [fit] What about those
# [fit]  other *functions*...

^ Adding additional packages to a lambda function means that you need to normally pip install them into a directory with the function zip it up and uploads (zip and upload to AWS lambda is just part of how apex works), but it's totally different than our normal virtualenv usage or apt-get package installs we're use too for deployment. It gets even stranger if your Dependency has to be compiled.

---

![fit, inline](lambda-packages.png)

# Lambda Packages

## Zappa Project/Gun.io

^ Thankfully! Those wicked sharp people in the Zappa Project already handled that for in Lambda packages. lambda-packages is a collection of several popular libraries already ready to be included in your lambda functions.

---

Lambda | Packages | List
--- | --- | ---
bcrypt | cffi | PyNaCl
datrie | LXML | misaka
MySQL-Python | numpy | OpenCV
Pillow (PIL) | psycopg2 | PyCrypto
cryptography   | pyproj | python-ldap
python-Levenshtein | regex |

^ And here is a quick look at some of those.

---

# Dependency Handling in Apex

```
"hooks":{
    "build": "pip install -r requirements.txt -t ."
}
```

^ Apex allows you to handle your dependencies in several ways including the typical requirements.txt file with hooks. Everytime you run apex deploy it will kick off this build hook prior to deploying your code to Amazon. This solves many of my problems, and for everything else... well... there is Stack GoogleFlow. Before we move on from functions let's talk about things to keep in mind while build functions.

---

# Function Considerations

- atomic
- idempotent

^ It's really important that you make your functions reasonably Atomic. I'm not going to go into the CS meaning of atomic here, but in general it means that a function can run in isolation from the rest of a larger and does a whole thing own it's on. This is what allows you to run multiple copies of a cloud function side by side for scaling or other purposes. Your functions also need to be idempotent, again skipping the CS view, it means if I run it twice for the same input produces the same output. This is important because many events, queues, streams, etc in the cloud world are guaranteed to fire at least once! so ... sometimes... when the cloud is gloomy it will send tell you the client uploaded the same file four times... in less than a second and you need to be able to handle that.

---

# Dis is the Remix

- Longer Jobs
- Legacy Pipelines

^ Cloud functions have limitations of execution time, Memory etc, and so we often can only sprinkle a bit of SERVERLESS! in... We might have some existing Luigi or Airflow pipelines we want to leverage or even just some straight up hack you did for that one time that you needed to get that data in place that has been in production for 4 years now. So let's look at a deeper example and explore some other possibilities.

---

# Hybrid Pipeline overview

![fit, inline](lambda-docker-1.png)

^ We can put some Moby on our longer/legacy pipelines, and ... That's just weird to say... We can put some docker on it... that's not accurate anymore is it... We can containerize our existing stuff and run it on ECS via task definitions, and we can launch those container tasks with a lambda in response to file upload event or cloudwatch scheduled events (aka the cloud crontab). They run and put the data into our data warehouse... This is grand, but running this in production gets fun fast... because containers die... they run out of memory, they get lost in cloud land, pandas explodes with some random garbage collection error.  So we need a way to watch it...

---
# Hybrid Pipeline overview

![fit, inline](lambda-docker-2.png)

^ Kinesis streams works well for this in AWS land. We can publish the task that we start to the stream, have ECS publish events for container state changes like starting, stopping, etc all into one place. We can also update our pipelines to publish events there as well. So we can no how far something got etc.

---
# Hybrid Pipeline overview

![fit, inline](lambda-docker-3.png)

^ We can watch that event stream with a lambda function (Kinesis will even invoke it for you! It's almost like the cloud wants you to do this...)  We could use this event stream handler with a multiphasic pipeline to watch for our first phase to end and kick off a second phase etc.

---
# Hybrid Pipeline overview

![fit, inline](lambda-docker-4.png)

^ I like to use it to grab events to record telemetry data for things how long it ran between each phase, overall, memory usage, anything we got an event for and we want to know about.

---
# Hybrid Pipeline overview

![fit, inline](lambda-docker-5.png)

^ Also Perlman forbid it doesn't send slack or pagerduty notifications. I normally farm that work out to a function as well. Especially for those things where my pipeline crashes in a way that I only get the ECS container stop event and not an error or event from my code running in the container.

---

# Closing thoughts

- Workload and Phases are important
- 14x improvement
- 0.73x improvement

^ I tried hard to come up with a good way to show comparisons between serverless! data pipelines and more traditional styles, and I had nifty numbers... but let's be real for a second. It completely depends on your work load and how much you can concurrently execute in your pipeline. I've seen it improve the processing of weather data streams by 14x or infrastructure metric data by 9x, but in those old slogs of wide data tables with lots transformations (22) and a ton of sequential phases it actually reduced the performance to 0.73X. That reduction came with greater visibility so it was worth it to them.

---

# [fit] Questions?

^ Does anyone have any questions... No okay thanks for your time and it was an honor to share this with you.
