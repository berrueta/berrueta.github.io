---
layout: post
title:  "SqsListener and threads"
date:   2025-08-30 05:49:29 +1200
category: software
tags:
  - java
  - aws
  - sqs
  - cloud
  - threads
---

There are multiple ways to consume messages from an AWS SQS queue in a Spring Boot application. One way to do it is to use the AWS SDK directly. It is a bit low level. It means you have to deal with polling the queue, handling the message, and deleting the message from the queue afterwards. And that is only the happy path!

Unless your application only has trivial scaling requirements, you probably
want to enable long polling to avoid excessive API calls (expensive and
inefficient). To improve throughput, you will need to implement batch
processing to retrieve and remove multiple messages from the queue with one
API call. And above all, you will need to set up a thread pool to process multiple messages in parallel and maximise the utilisation of the CPU.

On top of all that, you also need to initialise the AWS SDK client, which is often boilerplate code that you do not want to repeat in every application.

All of the above combined, that's a work to do, and it is easy to get it wrong. Fortunately, Spring Cloud AWS provides a much simpler way to consume messages from an SQS queue using the `@SqsListener` annotation. This library has been
around for a while, and if you gave it a try in the past, you may have been
left with mixed feelings. However, the library has improved a lot in recent years, and now it is a solid choice for consuming messages from SQS queues in a Spring Boot application.

## Basic usage

First, we need to add the dependency `io.awspring.cloud:spring-cloud-aws-starter-sqs`
to the project. Then we may need to configure some properties under
`spring.cloud.aws.*` in the `application.yml` file. This depends on how
our AWS credentials and region are set up. In some simple cases, the library
defaults will work out of the box.

...

## Scaling it up

...

## Starting and stopping the listener container

...