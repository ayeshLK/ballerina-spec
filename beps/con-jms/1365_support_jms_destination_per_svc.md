# Support `jms:Service` per Queue/Topic in JMS listener-service based message consumption

- Authors
  - Ayesh Almeida
- Reviewed by
  - Shafreen Anfar, Danesh Kuruppu, Thisaru Guruge
- Created date
  - 2025-06-26
- Issue
  - TBA
- State
  - Submitted

## Summary

This proposal introduces a design improvement to the Ballerina JMS listener service-based message consumption model by enabling **one JMS listener instance to serve multiple services**, each bound to its own **queue or topic**. Currently, a separate listener is required per queue/topic, limiting reusability and increasing configuration overhead. With this change, queue/topic-specific configurations along with session configurations and additional consumer configurations will be moved from the listener level to service-level annotations, resulting in a more scalable and modular design.

## Goals

* Enable the reuse of a single JMS listener across multiple services.
* Decouple queue/topic configuration from the listener and move it to service-level annotations.

## Motivation

In the current JMS implementation, each listener is tightly coupled to a single queue or topic. This design requires users to define multiple listeners even when all consumers can share the same JMS connection, resulting in:

* Increased configuration complexity.
* Redundant listener instances.
* Reduced runtime efficiency due to unnecessary connection creation.

By enabling **queue/topic per service** rather than per listener, we allow the developer to configure the queue/topic-specific behavior at the service level while keeping the connection setup at the listener level.

## Description

### Current Model:

```ballerina
listener jms:Listener jmsListener = check new ({
    connectionConfig: {
        initialContextFactory: "org.apache.activemq.jndi.ActiveMQInitialContextFactory", 
        providerUrl: "tcp://localhost:61616",
        connectionFactoryName: "ConnectionFactory"
    },
    acknowledgementMode: jms:AUTO_ACKNOWLEDGE,
    consumerOptions: {
        destination: {
            'type: jms:TOPIC,
            name: "test-topic"
        }
        ...
    }
});

service "consumer-service" on jmsListener {
    remote function onMessage(jms:Message message) { ... }
}
```

### Proposed Model:

We propose to **decouple the destination, session, and consumer configurations from the listener**, and move them into **service-level annotations**, enabling **a single listener to handle multiple JMS services**.

```ballerina
listener jms:Listener jmsListener = check new ({
  initialContextFactory: "org.apache.activemq.jndi.ActiveMQInitialContextFactory",
  providerUrl: "tcp://localhost:61616",
  connectionFactoryName: "ConnectionFactory"
});

@jms:ServiceConfig {
  sessionAckMode: jms:AUTO_ACKNOWLEDGE,
  config: {
    topicName: "topic-1"
    ...
  }
}
service "topic-1-consumer" on jmsListener {
  remote function onMessage(jms:Message message) { ... }
}

@jms:ServiceConfig {
  sessionAckMode: jms:AUTO_ACKNOWLEDGE,
  config: {
    topicName: "topic-2"
    ...
  }
}
service "topic-2-consumer" on jmsListener {
  remote function onMessage(jms:Message message) { ... }
}
```

* The destination details—such as the destination type (queue or topic) and the destination name (the actual name of the queue or topic)—are specified within the `jms:ServiceConfig` annotation. In this approach, the **service name does not need to match the destination name**.

### Key Changes

* To initialize the `jms:Listener`, the user should provide only the `jms:ConnectionConfiguration`. Hence, the listener `init` function will be updated as follows:

  ```ballerina
  public isolated function init(*jms:ConnectionConfiguration connectionConfig) returns Error?;
  ```

* Introduce a new service-level annotation `jms:ServiceConfig`.


  ```ballerina
  # Represents common configurations related to JMS destination.
  #
  # + messageSelector - Only messages with properties matching the message selector expression are added to the durable subscription. 
  #                     An empty string indicates that there is no message selector for the durable subscription.
  # + noLocal - If true then any messages published to the topic using this session's connection, or any other connection 
  #             with the same client identifier, will not be added to the durable subscription.
  type CommonDestinationConfig record {|
    string messageSelector = "";
    boolean noLocal = false;
  |};

  # Represents configurations for a JMS queue.
  #
  # + queueName - The name of the queue to consume messages from
  public type QueueConfig record {|
    *CommonDestinationConfig;
    string queueName;
  |};

  # Represents configurations for JMS topic subscription.
  #
  # + topicName - The name of the topic to subscribe to
  # + consumerType - The message consumer type
  # + subscriberName - the name used to identify the subscription
  public type TopicConfig record {|
    *CommonDestinationConfig;
    string topicName;
    jms:ConsumerType consumerType = jms:DEFAULT;
    string subscriberName?;
  |};

  # The service configuration type for the `jms:Service`.
  #
  # + acknowledgementMode - Configuration indicating how messages received by the session will be acknowledged
  # + config - The topic or queue configuration to subscribe to
  public type ServiceConfiguration record {|
      jms:AcknowledgementMode acknowledgementMode = jms:AUTO_ACKNOWLEDGE;
      QueueConfig|TopicConfig config;
  |};

  # Annotation to configure the `jms:Service`.
  public annotation ServiceConfiguration ServiceConfig on service;
  ```

## Testing

* Unit tests to validate the mapping of `@jms:ServiceConfig` to services.
* Integration tests for:
  * Multiple services on one listener with different queues/topics.
  * Invalid or missing `@jms:ServiceConfig`.
  * Conflicting configuration scenarios (e.g., two services with same destination name).
