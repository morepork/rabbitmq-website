<!--
Copyright (c) 2007-2019 Pivotal Software, Inc.

All rights reserved. This program and the accompanying materials
are made available under the terms of the under the Apache License,
Version 2.0 (the "License”); you may not use this file except in compliance
with the License. You may obtain a copy of the License at

https://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
-->
# RabbitMQ tutorial - Reliable Publishing with Publisher Confirms SUPPRESS-RHS

## Publisher Confirms

[Publisher confirms](/confirms.html#publisher-confirms) 
are a RabbitMQ extension to implement reliable
publishing. When publisher confirms are enabled on a channel,
messages the client publishes are confirmed asynchronously
by the broker, meaning they have been taken care of on the server
side.

### (using the Java client)

<xi:include href="site/tutorials/tutorials-help.xml.inc"/>

In this tutorial we're going to use publisher confirms to make
sure published messages have safely reached the broker. We will
cover several technics, each with their pros and cons.

### Enabling Publisher Confirms on a Channel

Publishers confirms are a RabbitMQ extension to the AMQP 0.9.1 standard,
so they are not enabled by default. Publisher confirms are
enabled at the channel level with the `confirmSelect` method:

<pre class="lang-java">
Channel channel = connection.createChannel();
channel.confirmSelect();
</pre>

Needless to say this call is mandatory each time you expect to use publisher
confirms.

### Publishing Messages Individually

Let's start with the simplest technics, that is publishing a message and waiting
synchronously for its confirmation:

<pre class="lang-java">
while(thereAreMessagesToPublish()) {
    byte[] body = ...;
    BasicProperties properties = ...;
    channel.basicPublish(exchange, queue, properties, body);
    channel.waitForConfirmsOrDie(5_000);
}
</pre>

In the previous sample, we publish a message as usual and wait for its
confirmation with the `Channel#waitForConfirmsOrDie(long)` method.
The method returns as soon as the message has been confirmed. If the
message is not confirmed within the timeout or if it is nack-ed (meaning
the broker could not take care of it for some reason), the method will
throw an exception. The handling of the exception will usually consists
in logging an error message and/or retrying to send the message. Note clients
have different ways to synchronously deal with publisher confirms, so make
sure to read carefully the documentation of the client you are using.

This technics is simple but has a major drawback: it is slow, 
as the confirmation of a message blocks the publishing
of all subsequent messages. Do not expect to scale to more than a few
hundreds of published messages per second. Nevertheless, this can be
enough for some applications.

> #### Are Publisher Confirms Asynchronous?
>
> We mentioned at the beginning that the broker confirms published
> messages asynchronously but in the first example the code waits
> synchronously until the message is confirmed. The client actually
> receives confirms asynchronously and unblocks the call to `waitForConfirmsOrDie`
> accordingly. Think of `waitForConfirmsOrDie` as a synchronous helper,
> whose internals are based on asynchronous notifications.


### Publishing Messages in Batch

To improve upon our previous example, we can publish a batch
of messages and wait for this whole batch to be confirmed.
The following example uses a batch of 100:

<pre class="lang-java">
int batchSize = 100;
int outstandingMessageCount = 0;
while(thereAreMessagesToPublish()) {
    byte[] body = ...;
    BasicProperties properties = ...;
    channel.basicPublish(exchange, queue, properties, body);
    outstandingMessageCount++;
    if (outstandingMessageCount == batchSize) {
        ch.waitForConfirmsOrDie(5_000);
        outstandingMessageCount = 0;
    }
}
if (outstandingMessageCount > 0) {
    ch.waitForConfirmsOrDie(5_000);
}
</pre>

Waiting for a batch of messages to be confirmed improves throughput drastically over
waiting for a confirm for individual message (up to 20-30 times with a remote RabbitMQ node).
One drawback is that we do not know exactly what went wrong in case of failure,
so we may have to keep a whole batch in memory to log something meaningul or
to re-publish the messages. And this solution is still synchronous, so it
blocks the publishing of messages.

### Handling Publisher Confirms Asynchronously

The broker confirms published messages asynchronously, one just needs
to register a callback on the client to be notified of these confirms:

<pre class="lang-java">
Channel channel = connection.createChannel();
channel.confirmSelect();
channel.addConfirmListener((sequenceNumber, multiple) -> {
    // code when message is confirmed
}, (sequenceNumber, multiple) -> {
    // code when message is nack-ed
});
</pre>

There are 2 callbacks: one for confirmed messages and one for nack-ed messages
(messages that can be considered lost by the broker). Each callback has
2 parameters:

 * sequence number: a number that identifies the confirmed
 or nack-ed message. We will see shortly how to correlate it with the published message.
 * multiple: this is a boolean value. If false, only one message is confirmed/nack-ed, if
 true, all messages with a lower or equal sequence number are confirmed/nack-ed.

The sequence number can be obtained with `Channel#getNextPublishSeqNo()`
before publishing:

<pre class="lang-java">
int sequenceNumber = channel.getNextPublishSeqNo());
ch.basicPublish(exchange, queue, properties, body);
</pre>

A simple way to correlate messages with sequence number consists 
in using a map. Let's assume we want to publish strings (easy
to turn into an array of bytes for publishing), here is a code
sample that uses a map to correlate the publishing sequence number
with the string body of the message:

<pre class="lang-java">
ConcurrentNavigableMap&lt;Long, String> outstandingConfirms = new ConcurrentSkipListMap&lt;>();
// ... code for confirm callbacks will come later
String body = "...";
outstandingConfirms.put(channel.getNextPublishSeqNo(), body);
channel.basicPublish(exchange, queue, properties, body.getBytes());
</pre>

The publishing code now tracks outbound messages with a map. We need
to clean this map when confirms arrive and do something like logging a warning
when messages are nack-ed:

<pre class="lang-java">
ConcurrentNavigableMap&lt;Long, String&gt; outstandingConfirms = new ConcurrentSkipListMap&lt;&gt;();
ConfirmCallback cleanOutstandingConfirms = (sequenceNumber, multiple) -> {
    if (multiple) {
        ConcurrentNavigableMap&lt;Long, String&gt; confirmed = outstandingConfirms.headMap(
          sequenceNumber, true
        );
        confirmed.clear();
    } else {
        outstandingConfirms.remove(sequenceNumber);
    }
};

channel.addConfirmListener(cleanOutstandingConfirms, (sequenceNumber, multiple) -> {
    String body = outstandingConfirms.get(sequenceNumber);
    System.err.format(
      "Message with body %s has been nack-ed. Sequence number: %d, multiple: %b%n", 
      body, sequenceNumber, multiple
    );
    cleanOutstandingConfirms.handle(sequenceNumber, multiple);
});
// ... publishing code
</pre>

The previous sample contains a callback that cleans the map when
confirms arrive. Note this callback handles both single and multiple
confirms. This callback is used when confirms arrive (as the first argument of
`Channel#addConfirmListener`). The callback for nack-ed messages
retrieves the message body and issue a warning. It then re-uses the
previous callback to clean the map of outstanding confirms (whether
messages are confirmed or nack-ed, their corresponding entries in the map
must be removed.)

> #### How to Track Outstanding Confirms?
>
> Our samples use a `ConcurrentNavigableMap` to track outstanding confirms.
> This data structure is convenient for several reasons. It allows to
> easily correlate a sequence number with a message (whatever the message data
> is) and to easily clean the entries up to a give sequence id (to handle
> multiple confirms/nacks). At last, it supports concurrent access, because
> confirm callbacks are called in a thread owned by the client library, which
> should be kept different from the publishing thread.
> 
> There are other ways to track outstanding confirms than with
> a sophisticated map implementation, like using a simple concurrent hash map
> and a variable to track the lower bound of the publishing sequence, but
> they are usually more involved and do not belong to a tutorial.

To sum up, handling publisher confirms asynchronously usually requires the
following steps:

 * provide a way to correlate the publishing sequence number with a message.
 * register a confirm listener on the channel to be notified when
 publisher acks/nacks arrive to perform the appropriate actions, like
 logging or re-publishing a nack-ed message. The sequence-number-to-message
 correlation mechanism may also require some cleaning during this step.
 * track the publishing sequence number before publishing a message.

> #### Re-publishing nack-ed Messages?
>
> It can be tempting to re-publish a nack-ed message from the corresponding
> callback but this should be avoided, as confirm callbacks are
> dispatched in an I/O thread from within channels are not supposed
> to do operations. A better solution consists in enqueuing the message in a in-memory
> queue which is polled by a publishing thread. A class like `ConcurrentLinkedQueue`
> would be a good candidate to transmit messages between the confirm callbacks
> and a publishing thread.

### Summary

Making sure published messages made it to the broker can be essential in some applications.
Publisher confirms are a RabbitMQ feature that helps to meet this requirement. Publisher
confirms are asynchronous in nature but it is also possible to handle them synchronously.
There is no definitive way to implement publisher confirms, this usually comes down
to the constraints in the application and in the overall system. Typical technics are:

 * publishing messages individually, waiting for the confirmation synchronously: simple, but very
 limited throughput.
 * publishing messages in batch, waiting for the confirmation synchronously for a batch: simple, reasonable
 throughput, but hard to reason about when something goes wrong.
 * asynchronous handling: best performance and use of resources, good control in case of error, but
 can be involved to implement correctly.

Putting it all together
-----------------------

The [`PublisherConfirms.java`](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/PublisherConfirms.java)
class contains code for the technics we covered. We can compile it, execute it as-is and
see how they each perform:

<pre class="lang-bash">
javac -cp $CP PublisherConfirms.java
java -cp $CP PublisherConfirms
</pre>

The output will look like the following:

<pre class="lang-bash">
Published 50,000 messages individually in 5,549 ms
Published 50,000 messages in batch in 2,331 ms
Published 50,000 messages and handled confirms asynchronously in 4,054 ms
</pre>

The output on your computer should look similar if the
client and the server sit on the same machine. Publishing messages individually
performs poorly as expected, but the results for asynchronously handling
are a bit disappointing compared to batch publishing.

Publisher confirms are very network-dependent, so we'd better off
trying with a remote node, which is more realistic as clients
and servers are usually not on the same machine in production.
`PublisherConfirms.java` can easily be changed to use a non-local node:

<pre class="lang-bash">
static Connection createConnection() throws Exception {
    ConnectionFactory cf = new ConnectionFactory();
    cf.setHost("remote-host");
    cf.setUsername("remote-user");
    cf.setPassword("remote-password");
    return cf.newConnection();
}
</pre>

Recompile the class, execute it again, and wait for the results:

<pre class="lang-bash">
Published 50,000 messages individually in 231,541 ms
Published 50,000 messages in batch in 7,232 ms
Published 50,000 messages and handled confirms asynchronously in 6,332 ms
</pre>

We see publishing individually now performs terribly. But
with the network between the client and the server, batch publishing and asynchronous handling
now perform similarly, with a small advantage for asynchronous handling of the publisher confirms.

Remember batch publishing is simple to implement, but do not make it easy to know
which message(s) could not make it to the broker in case of negative publisher acknowledgment.
Handling publisher confirms asynchronously is more involved to implement but provide
better granularity and better control over actions to perform when published messages
are nack-ed.
