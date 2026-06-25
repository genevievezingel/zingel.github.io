---
layout: post
title: "Quick look at AWS Lambda."
description: "AWS Lambda provides a new tool in your tool box for creating modular solutions. It allows independent execution of code based upon AWS events or time schedule."
date: 2016-01-10
read: "9 min read"
category: software
permalink: /blog/AWS-lambda/
---

>  "The times They are A changin'"

When Bob Dylan wrote these `famous words` “The times They are A changin”, back in 1964, he probably wasn't thinking about software development, however these words are applicable to modern software development.

The `change ` that is happening today is a movement away from mammoth applications towards meta solutions. These meta solutions are made up of lots of smaller independent components, with clear and distinct purpose, that work together to provide the application functionalities.

The web is constantly changing and we are faced with the harsh reality that any solution will be short lived and will require ` change`. It is easier and less risky to modify a single component, with a single responsibly, rather than the whole application.

The role of a developer has ` shifted ` to ensure that these components are working together and to **only** develop what is unique to the application.

### AWS Lambda
AWS Lambda provides a new tool in your` tool box` for creating modular solutions. It allows independent execution of code based upon AWS events or time schedule.

For example, if your application allows users to upload images to S3, which are then processed further, then this functionality could be move to AWS Lambda. 

#### Advantage:
There are two advantages:
- Reduces coupling as the code is  executed independently and is isolated from the existing application 
- AWS is responsible for the infrastructure and any scaling issues, whether it is executed once or one million times a second.

### Disadvantages

There are two disadvantages:
- The code has to finish executing within 5 minutes: an issue, if you were executing long running tasks i.e. video conversion.
- Any dependant libraries are loaded whenever  the code is executed. This increases the workload compared with  loading the libraries once .

### Example
The following example uses AWS Lambda to check whether a website is available.

### Background
I recently changed my webserver to  ` Puma` and I wanted a
quick and easy way to check whether my site was up.

### AWS Lambda Setup:
- Selected `Node.js` at runtime
- The handler `index.handler`
- The selected Role needs to have the correct permissions to execute the action

<img src="{{ '/assets/images/blog/aws_lambda/lambda1.png' | relative_url }}" alt="VSCode" itemprop="image"  class="u-photo">


#### Step 1.
I have used the inline editor to create the `node.js's javascript`,
but I could have automated the process by uploading the code as a file to `S3`. To test your Lambda function use the ` save and  test ` button to execute the function. The  log appears under the ` Monitoring tab `.
####  Step 2.
If the site is down the function sends a `notification email`. To do so, the `AWS user credentials` are used to authorise and to send an email. This user has to have the permission to send emails via SES (Amazon Simple Email Service).

The permission of this AWS user (setup in IAM)

~~~
		  {
			"Version": "2012-10-17",
			"Statement": [
			  {
				"Effect": "Allow",
				"Action": [
				  "ses:*"
				],
				"Resource": "*"
			  }
			]
		  }

~~~

<img src="{{ '/assets/images/blog/aws_lambda/lambda2.png' | relative_url }}" alt="VSCode" itemprop="image"  class="u-photo">


####  Step 3.
In our example we are using Node.js (Javascript). If you are using callbacks then it is important that you properly terminate your Lambda function otherwise the function will terminate when the
Node.js event queue is empty or your Lambda function times out. This additional time will add to the cost of using Lambda.

-<code> context.succeed(Object result) </code> notify Lambda that all callbacks have completed
successfully
-<code> context.fail(Error error);</code> is used to signal that a error
has occurred with the error JSON written to Cloud Watch.

<img src="{{ '/assets/images/blog/aws_lambda/lambda3.png' | relative_url }}" alt="VSCode" itemprop="image"  class="u-photo">


####  Step 4.
Event Source is the trigger that initiates the Lambda function. In my example, I am using ` Shedule Events ` to execute the Lambda function each hour.

There are a number of triggers that can be used including:
- S3
- Amazon Kinesis
- Amazon DynamoDB
- AWS CloudTrail
- Amazon API Gateway – allowing you to create a custom REST API
- Schedule Events


<img src="{{ '/assets/images/blog/aws_lambda/lambda4.png' | relative_url }}" alt="VSCode" itemprop="image"  class="u-photo">



####  Step 5.
To specify the right and privileges that are available to AWS Lambda is configured in the `configuration tag.` In my case, the role used by AWS Lambda required access to Lambda functions, and Cloud Watch. To configure roles go to <bold> IAM</bold> under ` Services `

~~~
		{
		  "Version": "2012-10-17",
		  "Statement": [
			{
			  "Effect": "Allow",
			  "Action": [
				"cloudwatch:*",
				"lambda:*",
				"logs:*"
			  ],
			  "Resource": "*"
			}
		  ]
		}

~~~

### Full Code Example

~~~
			var http = require('http');
			var aws = require('aws-sdk');
			var eParams;

			// Linking to Admin lambda

			var ses = new aws.SES({
				accessKeyId: ACCESS-KEY-ID,
				secretAccesskey: SECRET-ACCESSKEY,
				region: 'us-east-1'
			});

			exports.handler = function(event, context) {

				eParams = {
					Destination: {
						ToAddresses: ["chris@zingel.co.nz"]
					},
					Message: {
						Body: {
							Text: {
								Data: "Site is currently down"
							}
						},
						Subject: {
							Data: "Zingtech is Down"
						}
					},
					Source: "chris@waiwaia.nz"

				};

				function checkSiteUP(site) {
					return http.get({
							host: site,
							path: '/'
						}, function(response) {

						  if (response.statusCode !== 200) {
							var email = ses.sendEmail(eParams, function(err, data) {
							context.succeed('the site is down' + response.statusCode)});
						  } else {
							context.succeed('the site is up');
						  }
						});
			}
			checkSiteUP("zingtech.co.nz"); };
~~~

<h3> Inspiration </h3>
- <a href =" https://en.wikipedia.org/wiki/The_Times_They_Are_a-Changin%27_(song)"> The Times They Are a-Changin' </a>
- <a href ="http://docs.aws.amazon.com/lambda/latest/dg/welcome.html">AWS lambda Documentation</a>
