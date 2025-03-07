---
id: client-libraries-java
title: The Pulsar Java client
sidebar_label: Java
---

The Pulsar Java client can be used both to create Java producers, consumers, and [readers](#reader-interface) of messages and to perform [administrative tasks](admin-api-overview.md). The current version of the Java client is **{{pulsar:version}}**.

Javadoc for the Pulsar client is divided up into two domains, by package:

Package | Description | Maven Artifact
:-------|:------------|:--------------
[`org.apache.pulsar.client.api`](/api/client) | The producer and consumer API | [org.apache.pulsar:pulsar-client:{{pulsar:version}}](http://search.maven.org/#artifactdetails%7Corg.apache.pulsar%7Cpulsar-client%7C{{pulsar:version}}%7Cjar)
[`org.apache.pulsar.client.admin`](/api/admin) | The Java [admin API](admin-api-overview.md) | [org.apache.pulsar:pulsar-client-admin:{{pulsar:version}}](http://search.maven.org/#artifactdetails%7Corg.apache.pulsar%7Cpulsar-client-admin%7C{{pulsar:version}}%7Cjar)

This document will focus only on the client API for producing and consuming messages on Pulsar topics. For a guide to using the Java admin client, see [The Pulsar admin interface](admin-api-overview.md).

## Installation

The latest version of the Pulsar Java client library is available via [Maven Central](http://search.maven.org/#artifactdetails%7Corg.apache.pulsar%7Cpulsar-client%7C{{pulsar:version}}%7Cjar). To use the latest version, add the `pulsar-client` library to your build configuration.

### Maven

If you're using Maven, add this to your `pom.xml`:

```xml
<!-- in your <properties> block -->
<pulsar.version>{{pulsar:version}}</pulsar.version>

<!-- in your <dependencies> block -->
<dependency>
  <groupId>org.apache.pulsar</groupId>
  <artifactId>pulsar-client</artifactId>
  <version>${pulsar.version}</version>
</dependency>
```

### Gradle

If you're using Gradle, add this to your `build.gradle` file:

```groovy
def pulsarVersion = '{{pulsar:version}}'

dependencies {
    compile group: 'org.apache.pulsar', name: 'pulsar-client', version: pulsarVersion
}
```

## Connection URLs

To connect to Pulsar using client libraries, you need to specify a [Pulsar protocol](developing-binary-protocol.md) URL.

Pulsar protocol URLs are assigned to specific clusters, use the `pulsar` scheme and have a default port of 6650. Here's an example for `localhost`:

```http
pulsar://localhost:6650
```

If you have more than one broker, the URL may look like this:
```http
pulsar://localhost:6550,localhost:6651,localhost:6652
```

A URL for a production Pulsar cluster may look something like this:

```http
pulsar://pulsar.us-west.example.com:6650
```

If you're using [TLS](security-tls-authentication.md) authentication, the URL will look like something like this:

```http
pulsar+ssl://pulsar.us-west.example.com:6651
```

## Client 

You can instantiate a {@inject: javadoc:PulsarClient:/client/org/apache/pulsar/client/api/PulsarClient} object using just a URL for the target Pulsar [cluster](reference-terminology.md#cluster), like this:

```java
PulsarClient client = PulsarClient.builder()
        .serviceUrl("pulsar://localhost:6650")
        .build();
```

If you have multiple brokers, you can initiate a PulsarClient like this:
```java
PulsarClient client = PulsarClient.builder()
        .serviceUrl("pulsar://localhost:6650,localhost:6651,localhost:6652")
        .build();
```

> #### Default broker URLs for standalone clusters
> If you're running a cluster in [standalone mode](getting-started-standalone.md), the broker will be available at the `pulsar://localhost:6650` URL by default.

If you create a client, you may use the `loadConf` configuration. Below are the available parameters used in `loadConf`.

| Type | Name | <div style="width:260px">Description</div> | Default
|---|---|---|---
String | `serviceUrl` |Service URL provider for Pulsar service | None
String | `authPluginClassName` | Name of the authentication plugin | None
String | `authParams` | String represents parameters for the authentication plugin <br/><br/>**Example**<br/> key1:val1,key2:val2|None
long|`operationTimeoutMs`|Operation timeout |30000
long|`statsIntervalSeconds`|Interval between each stat info<br/><br/>Stats is activated with positive `statsInterval`<br/><br/>`statsIntervalSeconds` should be set to 1 second at least |60
int|`numIoThreads`| Number of threads used for handling connections to brokers | 1 
int|`numListenerThreads`|Number of threads used for handling message listeners | 1 
boolean|`useTcpNoDelay`|Whether to use TCP no-delay flag on the connection to disable Nagle algorithm |true
boolean |`useTls` |Whether to use TLS encryption on the connection| false
string | `tlsTrustCertsFilePath` |Path to the trusted TLS certificate file|None
boolean|`tlsAllowInsecureConnection`|Whether the Pulsar client accepts untrusted TLS certificate from broker | false
boolean | `tlsHostnameVerificationEnable` | Whether to enable TLS hostname verification|false
int|`concurrentLookupRequest`|Number of concurrent lookup requests allowed to send on each broker connection to prevent overload on broker|5000
int|`maxLookupRequest`|Maximum number of lookup requests allowed on each broker connection to prevent overload on broker | 50000
int|`maxNumberOfRejectedRequestPerConnection`|Maximum number of rejected requests of a broker in a certain time frame (30 seconds) after the current connection is closed and the client creates a new connection to connect to a different broker|50
int|`keepAliveIntervalSeconds`|Seconds of keeping alive interval for each client broker connection|30
int|`connectionTimeoutMs`|Duration of waiting for a connection to a broker to be established <br/><br/>If the duration passes without a response from a broker, the connection attempt is dropped|10000
int|`requestTimeoutMs`|Maximum duration for completing a request |60000
int|`defaultBackoffIntervalNanos`| Default duration for a backoff interval | TimeUnit.MILLISECONDS.toNanos(100);
long|`maxBackoffIntervalNanos`|Maximum duration for a backoff interval|TimeUnit.SECONDS.toNanos(30)

Check out the Javadoc for the {@inject: javadoc:PulsarClient:/client/org/apache/pulsar/client/api/PulsarClient} class for a full listing of configurable parameters.

> In addition to client-level configuration, you can also apply [producer](#configuring-producers) and [consumer](#configuring-consumers) specific configuration, as you'll see in the sections below.

## Producer

In Pulsar, producers write messages to topics. Once you've instantiated a {@inject: javadoc:PulsarClient:/client/org/apache/pulsar/client/api/PulsarClient} object (as in the section [above](#client-configuration)), you can create a {@inject: javadoc:Producer:/client/org/apache/pulsar/client/api/Producer} for a specific Pulsar [topic](reference-terminology.md#topic).

```java
Producer<byte[]> producer = client.newProducer()
        .topic("my-topic")
        .create();

// You can then send messages to the broker and topic you specified:
producer.send("My message".getBytes());
```

By default, producers produce messages that consist of byte arrays. You can produce different types, however, by specifying a message [schema](#schemas).

```java
Producer<String> stringProducer = client.newProducer(Schema.STRING)
        .topic("my-topic")
        .create();
stringProducer.send("My message");
```

> You should always make sure to close your producers, consumers, and clients when they are no longer needed:
> ```java
> producer.close();
> consumer.close();
> client.close();
> ```
>
> Close operations can also be asynchronous:
> ```java
> producer.closeAsync()
>    .thenRun(() -> System.out.println("Producer closed"));
>    .exceptionally((ex) -> {
>        System.err.println("Failed to close producer: " + ex);
>        return ex;
>    });
> ```

### Configure producer

If you instantiate a `Producer` object specifying only a topic name, as in the example above, the producer uses the default configuration. 

If you create a producer, you may use the `loadConf` configuration. Below are the available parameters used in `loadConf`.

Type | Name| <div style="width:300px">Description</div>|  Default
|---|---|---|---
String|	`topicName`|	Topic name| null|
String|`producerName`|Producer name| null
long|`sendTimeoutMs`|Message send timeout in ms.<br/><br/>If a message is not acknowledged by a server before the `sendTimeout` expires, an error is triggered.|30000
boolean|`blockIfQueueFull`|If set to `true`, when the outgoing message queue is full, the `Send` and `SendAsync` methods of producer block rather than failing and throwing errors. <br/><br>If set to `false`, when the outgoing message queue is full, the `Send` and `SendAsync` methods of producer fail and throw `ProducerQueueIsFullError` exceptions.<br/><br/>The size of the outgoing message queue is determined by the `MaxPendingMessages` parameter.|false
int|`maxPendingMessages`|Maximum size of a queue holding pending messages.<br/><br/>For example, a message waiting to receive an acknowledgment from a [broker](reference-terminology.md#broker). <br/><br/>By default, when the queue is full, all calls to the `Send` and `SendAsync` methods fail **unless** `BlockIfQueueFull` is set to `true`.|1000
int|`maxPendingMessagesAcrossPartitions`|Maximum number of pending messages across partitions. <br/><br/>This setting is used to lower the max pending messages for each partition ({@link #setMaxPendingMessages(int)}) if the total exceeds the configured value.|50000
MessageRoutingMode|`messageRoutingMode`|Message routing logic for producers on [partitioned topics](concepts-architecture-overview.md#partitioned-topics).<br/><br/> This logic is applied only when no key is set on messages. <br/><br/>Below are the available options: <br/><br/><li>`pulsar.RoundRobinDistribution`: round robin<br/><br/> <li>`pulsar.UseSinglePartition`: publish all messages to a single partition<br/><br/><li>`pulsar.CustomPartition`: a custom partitioning scheme|`pulsar.RoundRobinDistribution`
HashingScheme|`hashingScheme`|Hashing function that determines the partition on which a particular message is published (**partitioned topics only**).<br/><br/>Below are the available options:<br/><br/><li> `pulsar.JavaStringHash`: the equivalent of `String.hashCode()` in Java<br/><br/><li> `pulsar.Murmur3_32Hash`: applies the [Murmur3](https://en.wikipedia.org/wiki/MurmurHash) hashing function<br/><br/><li>`pulsar.BoostHash`: applies the hashing function from C++'s [Boost](https://www.boost.org/doc/libs/1_62_0/doc/html/hash.html) library |`HashingScheme.JavaStringHash`
ProducerCryptoFailureAction|`cryptoFailureAction`|Producer should take action when encryption fails.<br/><br/><li>**FAIL**: if encryption fails, unencrypted messages fail to send.</li><br/><li> **SEND**: if encryption fails, unencrypted messages are sent. |`ProducerCryptoFailureAction.FAIL`
long|`batchingMaxPublishDelayMicros`|Time period within which messages sent will be batched.|TimeUnit.MILLISECONDS.toMicros(1)
int|batchingMaxMessages|Maximum number of messages permitted in a batch.|1000
boolean|`batchingEnabled`|Enable batching of messages. |true
CompressionType|`compressionType`|Message data compression type used by a producer. <br/><br/>Below are the available options:<li>[`LZ4`](https://github.com/lz4/lz4)<br/><li>[`ZLIB`](https://zlib.net/)<br/><li>[`ZSTD`](https://facebook.github.io/zstd/)<br/><li>[`SNAPPY`](https://google.github.io/snappy/)| No compression

To use a non-default configuration, there's a variety of configurable parameters that you can set. 

For a full listing, see the Javadoc for the {@inject: javadoc:ProducerBuilder:/client/org/apache/pulsar/client/api/ProducerBuilder} class. Here's an example:

```java
Producer<byte[]> producer = client.newProducer()
    .topic("my-topic")
    .batchingMaxPublishDelay(10, TimeUnit.MILLISECONDS)
    .sendTimeout(10, TimeUnit.SECONDS)
    .blockIfQueueFull(true)
    .create();
```

### Message routing

When using partitioned topics, you can specify the routing mode whenever you publish messages using a producer. For more on specifying a routing mode using the Java client, see the [Partitioned Topics](cookbooks-partitioned.md) cookbook.

### Async send

You can also publish messages [asynchronously](concepts-messaging.md#send-modes) using the Java client. With async send, the producer will put the message in a blocking queue and return immediately. The client library will then send the message to the broker in the background. If the queue is full (max size configurable), the producer could be blocked or fail immediately when calling the API, depending on arguments passed to the producer.

Here's an example async send operation:

```java
producer.sendAsync("my-async-message".getBytes()).thenAccept(msgId -> {
    System.out.printf("Message with ID %s successfully sent", msgId);
});
```

As you can see from the example above, async send operations return a {@inject: javadoc:MessageId:/client/org/apache/pulsar/client/api/MessageId} wrapped in a [`CompletableFuture`](http://www.baeldung.com/java-completablefuture).

### Configure messages

In addition to a value, it's possible to set additional items on a given message:

```java
producer.newMessage()
    .key("my-message-key")
    .value("my-async-message".getBytes())
    .property("my-key", "my-value")
    .property("my-other-key", "my-other-value")
    .send();
```

As for the previous case, it's also possible to terminate the builder chain with `sendAsync()` and
get a future returned.

## Consumer

In Pulsar, consumers subscribe to topics and handle messages that producers publish to those topics. You can instantiate a new [consumer](reference-terminology.md#consumer) by first instantiating a {@inject: javadoc:PulsarClient:/client/org/apache/pulsar/client/api/PulsarClient} object and passing it a URL for a Pulsar broker (as [above](#client-configuration)).

Once you've instantiated a {@inject: javadoc:PulsarClient:/client/org/apache/pulsar/client/api/PulsarClient} object, you can create a {@inject: javadoc:Consumer:/client/org/apache/pulsar/client/api/Consumer} by specifying a [topic](reference-terminology.md#topic) and a [subscription](concepts-messaging.md#subscription-modes).

```java
Consumer consumer = client.newConsumer()
        .topic("my-topic")
        .subscriptionName("my-subscription")
        .subscribe();
```

The `subscribe` method will automatically subscribe the consumer to the specified topic and subscription. One way to make the consumer listen on the topic is to set up a `while` loop. In this example loop, the consumer listens for messages, prints the contents of any message that's received, and then [acknowledges](reference-terminology.md#acknowledgment-ack) that the message has been processed. If the processing logic fails, we use [negative acknowledgement](reference-terminology.md#acknowledgment-ack)
to have the message redelivered at a later point in time.

```java
while (true) {
  // Wait for a message
  Message msg = consumer.receive();

  try {
      // Do something with the message
      System.out.printf("Message received: %s", new String(msg.getData()));

      // Acknowledge the message so that it can be deleted by the message broker
      consumer.acknowledge(msg);
  } catch (Exception e) {
      // Message failed to process, redeliver later
      consumer.negativeAcknowledge(msg);
  }
}
```

### Configure consumer

If you instantiate a `Consumer` object specifying only a topic and subscription name, as in the example above, the consumer will use the default configuration. 

If you create a consumer, you may use the `loadConf` configuration. Below are the available parameters used in `loadConf`.

Type | Name| <div style="width:300px">Description</div>|  Default
|---|---|---|---
Set&lt;String&gt;|	`topicNames`|	Topic name|	Sets.newTreeSet()
Pattern|   `topicsPattern`|	Topic pattern	|None
String|	`subscriptionName`|	Subscription name|	None
SubscriptionType| `subscriptionType`|	Subscription type <br/><br/>There are three subscription types:<li>Exclusive</li><li>Failover</li><li>Shared</li>|SubscriptionType.Exclusive
int | `receiverQueueSize` | Size of a consumer's receiver queue. <br/><br/>For example, the number of messages that can be accumulated by a consumer before an application calls `Receive`. <br/><br/>A value higher than the default value increases consumer throughput, though at the expense of more memory utilization.| 1000
long|`acknowledgementsGroupTimeMicros`|Group a consumer acknowledgment for a specified time.<br/><br/>By default, a consumer uses 100ms grouping time to send out acknowledgments to a broker.<br/><br/>Setting a group time of 0 sends out acknowledgments immediately. <br/><br/>A longer ack group time is more efficient at the expense of a slight increase in message re-deliveries after a failure.|TimeUnit.MILLISECONDS.toMicros(100)
long|`negativeAckRedeliveryDelayMicros`|Delay to wait before redelivering messages that have failed to be process.<br/><br/> When an application uses {@link Consumer#negativeAcknowledge(Message)},   failed messages are redelivered after a fixed timeout. |TimeUnit.MINUTES.toMicros(1)
int |`maxTotalReceiverQueueSizeAcrossPartitions`|Max total receiver queue size across partitions.<br/><br/>This setting reduces the receiver queue size for individual partitions if the total receiver queue size exceeds this value.|50000
String|`consumerName`|Consumer name|null
long|`ackTimeoutMillis`|Timeout of unacked messages|0
long|`tickDurationMillis`|Granularity of the ack-timeout redelivery.<br/><br/>Using an higher `tickDurationMillis` reduces the memory overhead to track messages when the ack-timeout is set to a bigger value (for example, 1 hour).|1000
int|`priorityLevel`|Priority level for a consumer to which a broker gives more priority while dispatching messages in the shared subscription mode. <br/><br/>Here, the broker follows descending priorities. For example, 0=max-priority, 1, 2,...<br/><br/>In the shared subscription mode, the broker **first dispatches messages to the max priority level consumers if they have permits**. Otherwise, the broker considers next priority level consumers.<br/><br/> **Example 1**<br/><br/>If a subscription has consumerA with `priorityLevel` 0 and consumerB with `priorityLevel` 1, then the broker **only dispatches messages to consumerA until it runs out permits** and then starts dispatching messages to consumerB.<br/><br/>**Example 2**<br/><br/>Consumer Priority, Level, Permits<br/>C1, 0, 2<br/>C2, 0, 1<br/>C3, 0, 1<br/>C4, 1, 2<br/>C5, 1, 1<br/><br/>Order in which a broker dispatches messages to consumers is: C1, C2, C3, C1, C4, C5, C4.|0
ConsumerCryptoFailureAction|`cryptoFailureAction`|Consumer should take action when it receives a message that can not decrypt.<br/><br/><li>**FAIL**: this is the default option to fail messages until crypto succeeds.</li><br/><li> **DISCARD**: message is silently acknowledged and not delivered to an application.</li><br/><li>**CONSUME**: deliver encrypted messages to applications. It is the application's responsibility to decrypt the message.<br/><br/>If message are compressed, the decompression fails. <br/><br/>If messages contain batch messages, a client is not be able to retrieve individual messages in batch.<br/><br/>Delivered encrypted message contains {@link EncryptionContext} which contains encryption and compression information in it using which application can decrypt consumed message payload.|ConsumerCryptoFailureAction.FAIL</li>
SortedMap<String, String>|`properties`|A name or value property of this consumer.<br/><br/>`properties` is application defined metadata that can be attached to a consumer. <br/><br/>When getting a topic stats, this metadata is associated to the consumer stats for easier identification.|new TreeMap<>()
boolean|`readCompacted`|If `readCompacted` is enabled, a consumer reads messages from a compacted topic rather than reading a full message backlog of a topic.<br/><br/> This means if a topic has been compacted, a consumer only see the latest value for each key in the topic, up until the point in the topic message when backlog that has been compacted. Beyond that point, the messages are sent as normal.<br/><br/>`readCompacted` can only be enabled on subscriptions to persistent topics, which have a single active consumer (for example, failure or exclusive subscriptions). <br/><br/>Attempting to enable it on subscriptions to non-persistent topics or on shared subscriptions leads to a subscription call throwing a `PulsarClientException`.|false
SubscriptionInitialPosition|`subscriptionInitialPosition`|Initial position at which to set cursor when subscribing to a topic at first time.|SubscriptionInitialPosition.Latest
int|`patternAutoDiscoveryPeriod`|Topic auto discovery period when using a pattern for topic's consumer.<br/><br/>The default and minimum value is 1 minute.|1
RegexSubscriptionMode|`regexSubscriptionMode`|When subscribing to a topic using a regular expression, you can pick a certain type of topics.<br/><br/><li>**PersistentOnly**: only subscribe to persistent topics.</li><br/><li>**NonPersistentOnly**: only subscribe to non-persistent topics.</li><br/><li>**AllTopics**: subscribe to both persistent and non-persistent topics.</li>|RegexSubscriptionMode.PersistentOnly
DeadLetterPolicy|`deadLetterPolicy`|Dead letter policy for consumers.<br/><br/>By default, some messages are redelivered many times possible, even to the extent that it can be never stop.<br/><br/>By using the dead letter mechanism, messages have the max redelivery count. **When message exceeding the maximum number of redeliveries, messages are sent to the Dead Letter Topic and acknowledged automatically**.<br/><br/>You can enable the dead letter mechanism by setting `deadLetterPolicy`.<br/><br/>**Example**<br/><br/><code>client.newConsumer()<br/>.deadLetterPolicy(DeadLetterPolicy.builder().maxRedeliverCount(10).build())<br/>.subscribe();</code><br/><br/>Default dead letter topic name is `{TopicName}-{Subscription}-DLQ`.<br/><br/>To set a custom dead letter topic name:<br/><code>client.newConsumer()<br/>.deadLetterPolicy(DeadLetterPolicy.builder().maxRedeliverCount(10)<br/>.deadLetterTopic("your-topic-name").build())<br/>.subscribe();</code><br/><br/>When the dead letter policy is specified and no `ackTimeoutMillis` is specified, then the ack timeout is set to 30000 millisecond.|None
boolean|`autoUpdatePartitions`|If `autoUpdatePartitions` is enabled, a consumer subscribes to partition increasement automatically.<br/><br/>**Note**: this is only for partitioned consumers.|true
boolean|`replicateSubscriptionState`|If `replicateSubscriptionState` is enabled, a subscription state is replicated to geo-replicated clusters.|false

To use a non-default configuration, there's a variety of configurable parameters that you can set. For a full listing, see the Javadoc for the {@inject: javadoc:ConsumerBuilder:/client/org/apache/pulsar/client/api/ConsumerBuilder} class. Here's an example:

Here's an example configuration:

```java
Consumer consumer = client.newConsumer()
        .topic("my-topic")
        .subscriptionName("my-subscription")
        .ackTimeout(10, TimeUnit.SECONDS)
        .subscriptionType(SubscriptionType.Exclusive)
        .subscribe();
```

### Async receive

The `receive` method will receive messages synchronously (the consumer process will be blocked until a message is available). You can also use [async receive](concepts-messaging.md#receive-modes), which will return immediately with a [`CompletableFuture`](http://www.baeldung.com/java-completablefuture) object that completes once a new message is available.

Here's an example:

```java
CompletableFuture<Message> asyncMessage = consumer.receiveAsync();
```

Async receive operations return a {@inject: javadoc:Message:/client/org/apache/pulsar/client/api/Message} wrapped inside of a [`CompletableFuture`](http://www.baeldung.com/java-completablefuture).

### Multi-topic subscriptions

In addition to subscribing a consumer to a single Pulsar topic, you can also subscribe to multiple topics simultaneously using [multi-topic subscriptions](concepts-messaging.md#multi-topic-subscriptions). To use multi-topic subscriptions you can supply either a regular expression (regex) or a `List` of topics. If you select topics via regex, all topics must be within the same Pulsar namespace.

Here are some examples:

```java
import org.apache.pulsar.client.api.Consumer;
import org.apache.pulsar.client.api.PulsarClient;

import java.util.Arrays;
import java.util.List;
import java.util.regex.Pattern;

ConsumerBuilder consumerBuilder = pulsarClient.newConsumer()
        .subscriptionName(subscription);

// Subscribe to all topics in a namespace
Pattern allTopicsInNamespace = Pattern.compile("persistent://public/default/.*");
Consumer allTopicsConsumer = consumerBuilder
        .topicsPattern(allTopicsInNamespace)
        .subscribe();

// Subscribe to a subsets of topics in a namespace, based on regex
Pattern someTopicsInNamespace = Pattern.compile("persistent://public/default/foo.*");
Consumer allTopicsConsumer = consumerBuilder
        .topicsPattern(someTopicsInNamespace)
        .subscribe();
```

You can also subscribe to an explicit list of topics (across namespaces if you wish):

```java
List<String> topics = Arrays.asList(
        "topic-1",
        "topic-2",
        "topic-3"
);

Consumer multiTopicConsumer = consumerBuilder
        .topics(topics)
        .subscribe();

// Alternatively:
Consumer multiTopicConsumer = consumerBuilder
        .topics(
            "topic-1",
            "topic-2",
            "topic-3"
        )
        .subscribe();
```

You can also subscribe to multiple topics asynchronously using the `subscribeAsync` method rather than the synchronous `subscribe` method. Here's an example:

```java
Pattern allTopicsInNamespace = Pattern.compile("persistent://public/default.*");
consumerBuilder
        .topics(topics)
        .subscribeAsync()
        .thenAccept(this::receiveMessageFromConsumer);

private void receiveMessageFromConsumer(Consumer consumer) {
    consumer.receiveAsync().thenAccept(message -> {
                // Do something with the received message
                receiveMessageFromConsumer(consumer);
            });
}
```

### Subscription modes

Pulsar has various [subscription modes](concepts-messaging#subscription-modes) to match different scenarios. A topic can have multiple subscriptions with different subscription modes. However, a subscription can only have one subscription mode at a time.

A subscription is identified with the subscription name, and a subscription name can specify only one subscription mode at a time. You can change the subscription mode, yet you have to let all existing consumers of this subscription offline first.

Different subscription modes have different message distribution modes. This section describes the differences of subscription modes and how to use them.

In order to better describe their differences, assuming you have a topic named "my-topic", and the producer has published 10 messages.

```java
Producer<String> producer = client.newProducer(Schema.STRING)
        .topic("my-topic")
        .enableBatch(false)
        .create();
// 3 messages with "key-1", 3 messages with "key-2", 2 messages with "key-3" and 2 messages with "key-4"
producer.newMessage().key("key-1").value("message-1-1").send();
producer.newMessage().key("key-1").value("message-1-2").send();
producer.newMessage().key("key-1").value("message-1-3").send();
producer.newMessage().key("key-2").value("message-2-1").send();
producer.newMessage().key("key-2").value("message-2-2").send();
producer.newMessage().key("key-2").value("message-2-3").send();
producer.newMessage().key("key-3").value("message-3-1").send();
producer.newMessage().key("key-3").value("message-3-2").send();
producer.newMessage().key("key-4").value("message-4-1").send();
producer.newMessage().key("key-4").value("message-4-2").send();
```

#### Exclusive

Create a new consumer and subscribe with the `Exclusive` subscription mode.

```java
Consumer consumer = client.newConsumer()
        .topic("my-topic")
        .subscriptionName("my-subscription")
        .subscriptionType(SubscriptionType.Exclusive)
        .subscribe()
```

Only the first consumer is allowed to the subscription, other consumers receive an error. The first consumer receives all 10 messages, and the consuming order is the same as the producing order.

> Note:
>
> If topic is a partitioned topic, the first consumer subscribes to all partitioned topics, other consumers are not assigned with partitions and receive an error. 

#### Failover

Create new consumers and subscribe with the`Failover` subscription mode.

```java
Consumer consumer1 = client.newConsumer()
        .topic("my-topic")
        .subscriptionName("my-subscription")
        .subscriptionType(SubscriptionType.Failover)
        .subscribe()
Consumer consumer2 = client.newConsumer()
        .topic("my-topic")
        .subscriptionName("my-subscription")
        .subscriptionType(SubscriptionType.Failover)
        .subscribe()
//conumser1 is the active consumer, consumer2 is the standby consumer.
//consumer1 receives 5 messages and then crashes, consumer2 takes over as an  active consumer.

  
```

Multiple consumers can attach to the same subscription, yet only the first consumer is active, and others are standby. When the active consumer is disconnected, messages will be dispatched to one of standby consumers, and the standby consumer becomes active consumer. 

If the first active consumer receives 5 messages and is disconnected, the standby consumer becomes active consumer. Consumer1 will receive:

```
("key-1", "message-1-1")
("key-1", "message-1-2")
("key-1", "message-1-3")
("key-2", "message-2-1")
("key-2", "message-2-2")
```

consumer2 will receive:

```
("key-2", "message-2-3")
("key-3", "message-3-1")
("key-3", "message-3-2")
("key-4", "message-4-1")
("key-4", "message-4-2")
```

> Note:
>
> If a topic is a partitioned topic, each partition only has one active consumer, messages of one partition only distributed to one consumer, messages of multiple partitions are distributed to multiple consumers. 

#### Shared

Create new consumers and subscribe with `Shared` subscription mode:

```java
Consumer consumer1 = client.newConsumer()
        .topic("my-topic")
        .subscriptionName("my-subscription")
        .subscriptionType(SubscriptionType.Shared)
        .subscribe()
  
Consumer consumer2 = client.newConsumer()
        .topic("my-topic")
        .subscriptionName("my-subscription")
        .subscriptionType(SubscriptionType.Shared)
        .subscribe()
//Both consumer1 and consumer 2 is active consumers.
```

In shared subscription mode, multiple consumers can attach to the same subscription and message are delivered in a round robin distribution across consumers.

If a broker dispatches only one message at a time, consumer1 will receive:

```
("key-1", "message-1-1")
("key-1", "message-1-3")
("key-2", "message-2-2")
("key-3", "message-3-1")
("key-4", "message-4-1")
```

consumer 2 will receive:

```
("key-1", "message-1-2")
("key-2", "message-2-1")
("key-2", "message-2-3")
("key-3", "message-3-2")
("key-4", "message-4-2")
```

`Shared` subscription is different from `Exclusive` and `Failover` subscription modes. `Shared` subscription has better flexibility, but cannot provide order guarantee.

#### Key_shared

This is a new subscription mode since 2.4.0 release, create new consumers and subscribe with `Key_Shared` subscription mode:

```java
Consumer consumer1 = client.newConsumer()
        .topic("my-topic")
        .subscriptionName("my-subscription")
        .subscriptionType(SubscriptionType.Key_Shared)
        .subscribe()
  
Consumer consumer2 = client.newConsumer()
        .topic("my-topic")
        .subscriptionName("my-subscription")
        .subscriptionType(SubscriptionType.Key_Shared)
        .subscribe()
//Both consumer1 and consumer2 are active consumers.
```

`Key_Shared` subscription is like `Shared` subscription, all consumers can attach to the same subscription. But it is different from `Key_Shared` subscription, messages with the same key are delivered to only one consumer in order. The possible distribution of messages between different consumers(by default we do not know in advance which keys will be assigned to a consumer, but a key will only be assigned to a consumer at the same time. ) .

consumer1 will receive:

```
("key-1", "message-1-1")
("key-1", "message-1-2")
("key-1", "message-1-3")
("key-3", "message-3-1")
("key-3", "message-3-2")
```

consumer 2 will receive:

```
("key-2", "message-2-1")
("key-2", "message-2-2")
("key-2", "message-2-3")
("key-4", "message-4-1")
("key-4", "message-4-2")
```

> Note:
>
> If the message key is not specified, messages without key will be dispatched to one consumer in order by default.

## Reader 

With the [reader interface](concepts-clients.md#reader-interface), Pulsar clients can "manually position" themselves within a topic, reading all messages from a specified message onward. The Pulsar API for Java enables you to create  {@inject: javadoc:Reader:/client/org/apache/pulsar/client/api/Reader} objects by specifying a topic, a {@inject: javadoc:MessageId:/client/org/apache/pulsar/client/api/MessageId}, and {@inject: javadoc:ReaderConfiguration:/client/org/apache/pulsar/client/api/ReaderConfiguration}.

Here's an example:

```java
ReaderConfiguration conf = new ReaderConfiguration();
byte[] msgIdBytes = // Some message ID byte array
MessageId id = MessageId.fromByteArray(msgIdBytes);
Reader reader = pulsarClient.newReader()
        .topic(topic)
        .startMessageId(id)
        .create();

while (true) {
    Message message = reader.readNext();
    // Process message
}
```

In the example above, a `Reader` object is instantiated for a specific topic and message (by ID); the reader then iterates over each message in the topic after the message identified by `msgIdBytes` (how that value is obtained depends on the application).

The code sample above shows pointing the `Reader` object to a specific message (by ID), but you can also use `MessageId.earliest` to point to the earliest available message on the topic of `MessageId.latest` to point to the most recent available message.

## Schema

In Pulsar, all message data consists of byte arrays "under the hood." [Message schemas](concepts-schema-registry.md) enable you to use other types of data when constructing and handling messages (from simple types like strings to more complex, application-specific types). If you construct, say, a [producer](#producers) without specifying a schema, then the producer can only produce messages of type `byte[]`. Here's an example:

```java
Producer<byte[]> producer = client.newProducer()
        .topic(topic)
        .create();
```

The producer above is equivalent to a `Producer<byte[]>` (in fact, you should *always* explicitly specify the type). If you'd like to use a producer for a different type of data, you'll need to specify a **schema** that informs Pulsar which data type will be transmitted over the [topic](reference-terminology.md#topic).

### Schema example

Let's say that you have a `SensorReading` class that you'd like to transmit over a Pulsar topic:

```java
public class SensorReading {
    public float temperature;

    public SensorReading(float temperature) {
        this.temperature = temperature;
    }

    // A no-arg constructor is required
    public SensorReading() {
    }

    public float getTemperature() {
        return temperature;
    }

    public void setTemperature(float temperature) {
        this.temperature = temperature;
    }
}
```

You could then create a `Producer<SensorReading>` (or `Consumer<SensorReading>`) like so:

```java
Producer<SensorReading> producer = client.newProducer(JSONSchema.of(SensorReading.class))
        .topic("sensor-readings")
        .create();
```

The following schema formats are currently available for Java:

* No schema or the byte array schema (which can be applied using `Schema.BYTES`):

  ```java
  Producer<byte[]> bytesProducer = client.newProducer(Schema.BYTES)
        .topic("some-raw-bytes-topic")
        .create();
  ```

  Or, equivalently:

  ```java
  Producer<byte[]> bytesProducer = client.newProducer()
        .topic("some-raw-bytes-topic")
        .create();
  ```

* `String` for normal UTF-8-encoded string data. This schema can be applied using `Schema.STRING`:

  ```java
  Producer<String> stringProducer = client.newProducer(Schema.STRING)
        .topic("some-string-topic")
        .create();
  ```

* JSON schemas can be created for POJOs using `Schema.JSON`. Here's an example:

  ```java
  Producer<MyPojo> pojoProducer = client.newProducer(Schema.JSON(MyPojo.class))
        .topic("some-pojo-topic")
        .create();
  ```

* Protobuf schemas can be generate using `Schema.PROTOBUF`. The following example shows how to create the Protobuf schema and use it to instantiate a new producer:

  ```java
  Producer<MyProtobuf> protobufProducer = client.newProducer(Schema.PROTOBUF(MyProtobuf.class))
        .topic("some-protobuf-topic")
        .create();
  ```

* Avro schemas can be defined with the help of `Schema.AVRO`. The next code snippet demonstrates the creation and usage of the Avro schema:
  
  ```java
  Producer<MyAvro> avroProducer = client.newProducer(Schema.AVRO(MyAvro.class))
        .topic("some-avro-topic")
        .create();
  ```

## Authentication

Pulsar currently supports two authentication schemes: [TLS](security-tls-authentication.md) and [Athenz](security-athenz.md). The Pulsar Java client can be used with both.

### TLS Authentication

To use [TLS](security-tls-authentication.md), you need to set TLS to `true` using the `setUseTls` method, point your Pulsar client to a TLS cert path, and provide paths to cert and key files.

Here's an example configuration:

```java
Map<String, String> authParams = new HashMap<>();
authParams.put("tlsCertFile", "/path/to/client-cert.pem");
authParams.put("tlsKeyFile", "/path/to/client-key.pem");

Authentication tlsAuth = AuthenticationFactory
        .create(AuthenticationTls.class.getName(), authParams);

PulsarClient client = PulsarClient.builder()
        .serviceUrl("pulsar+ssl://my-broker.com:6651")
        .enableTls(true)
        .tlsTrustCertsFilePath("/path/to/cacert.pem")
        .authentication(tlsAuth)
        .build();
```

### Athenz

To use [Athenz](security-athenz.md) as an authentication provider, you need to [use TLS](#tls-authentication) and provide values for four parameters in a hash:

* `tenantDomain`
* `tenantService`
* `providerDomain`
* `privateKey`

You can also set an optional `keyId`. Here's an example configuration:

```java
Map<String, String> authParams = new HashMap<>();
authParams.put("tenantDomain", "shopping"); // Tenant domain name
authParams.put("tenantService", "some_app"); // Tenant service name
authParams.put("providerDomain", "pulsar"); // Provider domain name
authParams.put("privateKey", "file:///path/to/private.pem"); // Tenant private key path
authParams.put("keyId", "v1"); // Key id for the tenant private key (optional, default: "0")

Authentication athenzAuth = AuthenticationFactory
        .create(AuthenticationAthenz.class.getName(), authParams);

PulsarClient client = PulsarClient.builder()
        .serviceUrl("pulsar+ssl://my-broker.com:6651")
        .enableTls(true)
        .tlsTrustCertsFilePath("/path/to/cacert.pem")
        .authentication(athenzAuth)
        .build();
```

> #### Supported pattern formats
> The `privateKey` parameter supports the following three pattern formats:
> * `file:///path/to/file`
> * `file:/path/to/file`
> * `data:application/x-pem-file;base64,<base64-encoded value>`
