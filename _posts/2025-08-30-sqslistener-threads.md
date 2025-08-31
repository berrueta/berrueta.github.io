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

There are multiple ways to consume messages from an AWS SQS queue in a Spring Boot application. One way to do it is to use the AWS SDK directly. This is a low level API, and that means you have to deal with polling the queue, handling the message, and deleting the message from the queue afterwards. And that is just for the happy path!

Unless your application only has trivial scaling requirements, you probably
want to enable long polling to avoid excessive API calls (expensive and
inefficient). To improve throughput, you will need to implement batch
processing to retrieve and remove multiple messages from the queue with one
API call. And above all, you will need to set up a thread pool to process multiple messages in parallel and maximise the utilisation of the CPU.

On top of all that, you also need to initialise the AWS SDK client, which is often boilerplate code that you do not want to repeat in every application.

All of the above combined, that's a fair amount of scaffolding, and it is easy to get it wrong. Fortunately, Spring Cloud AWS provides a much simpler way to consume messages from an SQS queue using the `@SqsListener` annotation. This framework has been
around for a while, and if you gave it a try in the past, you may have been
left with mixed feelings. However, the framework has improved a lot in recent years, and now it is a solid choice for consuming messages from SQS queues in a Spring Boot application.

## Basic usage

First, we add the dependency `io.awspring.cloud:spring-cloud-aws-starter-sqs`
to the project. It may be necessary to configure some properties under
`spring.cloud.aws.*` in the `application.yml` file. This depends on how
our AWS credentials and region are set up. In some simple cases, the library
defaults will work out of the box.

The simplest case to consume SQS messages is to annotate a method with the
`@SqsListener` annotation and indicate the name of a queue:

```java
@SqsListener(value = TEST_QUEUE_NAME)
public void receiveMessage(String message) {
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

The simplistic method signature presented in the previous section may led us to believe
that the application will process messages one at a time, which sounds inefficient.

Let's do an experiment. The listener will introduce a delay of 0.1 seconds to simulate processing time,
and then it will log the message:

```java
@SqsListener(value = TEST_QUEUE_NAME)
public void receiveMessage(String message) throws InterruptedException {
    Thread.sleep(DELAY_PER_MESSAGE_MS);
    logger.info("Consumed message: {}", message);
}
```

If we insert 100 messages in the queue and then run the application,
we should expect that the application will need 10 seconds (100 * 0.1) to process all the messages. But instead, the
application finishes in roughly 1 second, which is much better. The logs tell
an interesting story:

```console
18:48:20.390+12:00 --- [QueueListener-5]  : Consumed message: [message 5]
18:48:20.390+12:00 --- [QueueListener-1]  : Consumed message: [message 1]
18:48:20.390+12:00 --- [QueueListener-4]  : Consumed message: [message 4]
18:48:20.390+12:00 --- [QueueListener-2]  : Consumed message: [message 2]
18:48:20.391+12:00 --- [QueueListener-8]  : Consumed message: [message 8]
18:48:20.392+12:00 --- [QueueListener-10] : Consumed message: [message 10]
18:48:20.394+12:00 --- [QueueListener-3]  : Consumed message: [message 3]
18:48:20.395+12:00 --- [QueueListener-6]  : Consumed message: [message 6]
18:48:20.395+12:00 --- [QueueListener-9]  : Consumed message: [message 9]
18:48:20.395+12:00 --- [QueueListener-7]  : Consumed message: [message 7]
18:48:20.543+12:00 --- [QueueListener-9]  : Consumed message: [message 14]
18:48:20.543+12:00 --- [QueueListener-3]  : Consumed message: [message 11]
18:48:20.543+12:00 --- [QueueListener-5]  : Consumed message: [message 12]
18:48:20.543+12:00 --- [QueueListener-7]  : Consumed message: [message 16]
...
```

That is interesting! Turns out that the application is not consuming the messages sequentially one by one as we may
assume from our simplistic listener method. Instead, note how the first
10 messages are consumed in parallel by 10 different threads named `QueueListener-1` to `QueueListener-10`. Then, once the
first 10 messages are processed, the threads became available to process more messages from the queue. Each one of those 10 threads ends up consuming 10 messages, and overall they get the job done in just 1 second.

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

The `maxConcurrentMessages` parameter essentially controls the size of the thread pool. That means the framework automatically enables our
application to process the messages concurrently, taking full advantage of the available resources, and we do not need
to do anything special to enable this behaviour. Even better: error handling is trivial, because our application does not
need to handle partial failure scenarios that are typical when manually processing batches.

## Beware of batch processing

Despite the simplicity and the convenience of the solution presented above,
which efficiently consumes messages in parallel with minimal effort, sometimes
it may be necessary for the application to deliberately receive messages in
batches. For example, this is useful when the application wants to query a
database to retrieve some data related to the messages, and it is more efficient
to do it in a single query for multiple messages rather than one query per
message.

This can be achieved by simply changing the method signature to receive a list
of messages:

```java
public void receiveMessages(List<String> messages) throws InterruptedException {
    for (String individualMessage : messages) {
        Thread.sleep(DELAY_PER_MESSAGE_MS);
    }
    logger.info("Consumed batch: {}", messages);
}
```

However, if we run this code, something surprising happens. Rather than taking
1 second to process all 100 messages, the application now takes roughly 10 seconds:

```console
21:18:49.932+12:00 --- [QueueListener-1] : Consumed batch: [message 1, message 2, message 3, message 4, message 5, message 6, message 7, message 8, message 9, message 10]
21:18:51.003+12:00 --- [QueueListener-2] : Consumed batch: [message 11, message 12, message 13, message 14, message 15, message 16, message 17, message 18, message 19, message 20]
21:18:52.070+12:00 --- [QueueListener-3] : Consumed batch: [message 21, message 22, message 23, message 24, message 25, message 26, message 27, message 28, message 29, message 30]
...
```

What is going on here? How come the application is so much slower when processing messages in
batches? This is exactly the opposite of what we were trying to achieve!

Ignore for a moment the fact that the first batch was smaller than the rest.
The logs indicate that the batches were consumed by different threads, but rather than
running in parallel, those threads processed the batches sequentially, with one batch not
starting until the previous batch finished. This is because the `maxConcurrentMessages`
parameter remains at 10. Therefore the framework fills each batch with up to 10 messages, but
because all of those messages are in flight, the framework will not use other threads to process
more messages until the previous batch is fully processed, because that would exceed the
limit of 10 concurrent messages at maximum.

There are a number of ways around that. One option is to increase the value of
`maxConcurrentMessages` to 100. That will enable the framework to utilise the same
10 threads that it was using before, but this time each thread will process a batch
of 10 messages, and the application will finish in roughly 1 second again.

Still, it is worth considering the trade-offs of this approach. We are still achieving
the same throughput as before, but now the application has more responsibilities. For
example, the application is now responsible for handling partial failures within a batch.
Therefore this approach is only recommended if the application really needs to process
messages in batches. Otherwise, it is better to stick to the simpler approach of
processing messages one by one.

## Starting and stopping the listener container

To conclude this post, let's see how the application can start and stop the listener
container. This is useful in tests, because we may want to add many messages to the
queue before the listener starts consuming them. It is also useful in production, for example if
the application needs to temporarily stop consuming messages.

By default, the listener container starts automatically when the application starts.
This can be changed by setting the property `spring.cloud.aws.sqs.listener.auto-startup` to `false`.

Then the application or the test can start and stop the listener container programmatically
by retrieving it from the `MessageListenerContainerRegistry` bean. This is where the
`id` parameter of the `@SqsListener` annotation comes into play. For example, in a Spring Boot
integration test:

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE)
// other annotations omitted
public class SqsConsumerIT {
    @Autowired
    private MessageListenerContainerRegistry messageListenerContainerRegistry;

    @Test
    public void testConsumeMessages() {
        // add messages to the queue
        ...
        // start the listener container
        messageListenerContainerRegistry.getListenerContainer(TEST_QUEUE_LISTENER).start();
        // wait for messages to be consumed
        ...
        // stop the listener container
        messageListenerContainerRegistry.getListenerContainer(TEST_QUEUE_LISTENER).stop();
    }
}
```
