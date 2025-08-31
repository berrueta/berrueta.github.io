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

The simplest case is to annotate a method with the `@SqsListener` annotation and indicate the name of a queue:

```java
@SqsListener(value = TEST_QUEUE_NAME)
public void receiveMessages(String message) {
  ...
}
```

That's it. The framework takes care of polling SQS using long polling, and then feeds those messages
to our method. If the method completes successfully, the framework will delete the
message from the queue, but if the method throws, the framework will leave the message in the queue, and
depending on the configuration of the SQS queue, the message may be retried later or sent to a dead-letter queue (DLQ).

The method is not limited to receiving a `String`. The framework can deserialise JSON objects using
Jackson, in a similar fashion to how Spring MVC works, so we can have parameters of type `MyClass`. Another
option is to receive a parameter of type `Message<T>` like `Message<String>` or `Message<MyClass>`, which
allows the application to obtain some metadata about the incoming message. There are also a number of
other additional parameters that can be added to the method signature, for example `@Header("my-header") String headerValue`, `Acknowledgement` or `Visibility`. The latter two allow the application to manually control the acknowledgement
of the message from the queue, or change the visibiliy timeout if the application needs more time to process
the message.

## Scaling it up

The simplistic method signature presented in the previous section may suggest that the application will
only process one message at a time, which sounds inefficient.

Let's modify the method to do an experiment. The listener will introduce a delay of 0.1 seconds to simulate processing time,
and it will log the received message:

```java
@SqsListener(value = TEST_QUEUE_NAME)
public void receiveMessages(String message) throws InterruptedException {
    Thread.sleep(DELAY_PER_MESSAGE_MS);
    logger.info("Consumed message: {}", message);
}
```

If we introduce 100 messages in the queue and then run the application, and each one takes 0.1 seconds to process,
we should expect that the application will need 10 seconds (100 * 0.1) to process all the messages. But instead, the
application finishes in roughly 1 second, and we observe the following output:

```console
18:48:20.390+12:00  INFO 2144 --- [QueueListener-5] org.example.SqsConsumer                  : Consumed message: [message 5]
18:48:20.390+12:00  INFO 2144 --- [QueueListener-1] org.example.SqsConsumer                  : Consumed message: [message 1]
18:48:20.390+12:00  INFO 2144 --- [QueueListener-4] org.example.SqsConsumer                  : Consumed message: [message 4]
18:48:20.390+12:00  INFO 2144 --- [QueueListener-2] org.example.SqsConsumer                  : Consumed message: [message 2]
18:48:20.391+12:00  INFO 2144 --- [QueueListener-8] org.example.SqsConsumer                  : Consumed message: [message 8]
18:48:20.392+12:00  INFO 2144 --- [QueueListener-10] org.example.SqsConsumer                 : Consumed message: [message 10]
18:48:20.394+12:00  INFO 2144 --- [QueueListener-3] org.example.SqsConsumer                  : Consumed message: [message 3]
18:48:20.395+12:00  INFO 2144 --- [QueueListener-6] org.example.SqsConsumer                  : Consumed message: [message 6]
18:48:20.395+12:00  INFO 2144 --- [QueueListener-9] org.example.SqsConsumer                  : Consumed message: [message 9]
18:48:20.395+12:00  INFO 2144 --- [QueueListener-7] org.example.SqsConsumer                  : Consumed message: [message 7]
18:48:20.543+12:00  INFO 2144 --- [QueueListener-9] org.example.SqsConsumer                  : Consumed message: [message 14]
18:48:20.543+12:00  INFO 2144 --- [QueueListener-3] org.example.SqsConsumer                  : Consumed message: [message 11]
18:48:20.543+12:00  INFO 2144 --- [QueueListener-5] org.example.SqsConsumer                  : Consumed message: [message 12]
18:48:20.543+12:00  INFO 2144 --- [QueueListener-7] org.example.SqsConsumer                  : Consumed message: [message 16]
...
```

That is interesting! Turns out that the application is not consuming the messages sequentially one by one as we may
assume from our simplistic listener method. Instead, note how the first
10 messages were consumed in parallel by 10 different threads named `QueueListener-1` to `QueueListener-10`. Then, as those
first 10 messages were processed, those threads became available to process more messages from the queue. Each one of those 10 threads ends up consuming 10 messages, and overall they get the job done in just 1 second.

So clearly the framework is doing more than just invoking the listener method for each message. It is actually polling the messages from the SQS using a batch API call, and then passing each message to the listener method in parallel using a thread
pool. This becomes clearer if we modify the annotation to include the default values for some parameters:

```java
// see https://docs.awspring.io/spring-cloud-aws/docs/3.3.0/reference/html/index.html#sqs-integration for details about the parameters
@SqsListener(
        id = TEST_QUEUE_LISTENER, // we will use this later
        value = TEST_QUEUE_NAME,
        maxConcurrentMessages = "10", // default is 10
        maxMessagesPerPoll = "10", // default is 10, should be less or equal than the above
        pollTimeoutSeconds = "10" // default is 10 seconds
)
```

The `maxConcurrentMessages` parameter essentially controls the size of the thread pool. Great! The framework allows our
application to process the messages concurrently, taking full advantage of the available resources, and we do not need
to do anything special to enable this behaviour. Even better: error handling is trivial, because our application does not
need to handle partial failure scenarios that are typical when manually processing batches.

## Beware of batch processing

...

## Starting and stopping the listener container

...
