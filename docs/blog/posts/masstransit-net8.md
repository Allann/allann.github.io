---
title: MassTransit (pt1)
date: 2023-11-02
authors: [allann]
categories:
  - masstransit
  - rabbitmq
tags: [design]
---

# Utilizing MassTransit in .NET Core with RabbitMQ

Are you in search of an open-source distributed application framework designed for .NET, one that streamlines the development of message-based applications? MassTransit is the solution you've been seeking. It offers a straightforward yet robust means of constructing distributed systems using message-based architectural paradigms, including event-driven, publish/subscribe, and request/response patterns.

In the next few posts, I will explain my thoughts and guide you through the process of harnessing MassTransit within a .NET-based ASP.NET Core Web API. 

By the conclusion of this series, you will be well-equipped to establish and configure a producer and consumer message service for RabbitMQ. We will also delve into some advanced RabbitMQ use cases and explore key concepts essential to effectively work with distributed solutions.

<!-- more -->

## Prerequisites

Before we embark on this engaging endeavor, please take a moment to review the following prerequisites to ensure a smooth and productive learning experience:

- A development environment such as Rider, Visual Studio Code, Visual Studio IDE, or any other C# text editor.
- Proficiency in C# and a solid understanding of the .NET Framework.
- Familiarity with RabbitMQ.
- A project where you can implement both the producer and consumer components.
- Docker to host and execute RabbitMQ efficiently.

## An introduction to MassTransit

MassTransit stands as an open-source distributed application framework tailored for .NET, simplifying the development of message-based applications. It offers a straightforward yet potent avenue to construct distributed systems using message-based architectures like event-driven, publish/subscribe, and request/response patterns.

With MassTransit, developers gain the capability to build robust, scalable, and fault-tolerant distributed applications that can seamlessly communicate across diverse platforms and languages. It accommodates various messaging transports and protocols, including RabbitMQ, Azure Service Bus, Amazon Simple Queue Service (SQS), and Apache Kafka, allowing for smooth transitions between platforms.

MassTransit extends its utility with features like message serialization, message routing, message retry policies, and message monitoring. These attributes enable developers to create highly resilient and dependable distributed systems. Its appeal is evident.

In summary, MassTransit stands as a potent and adaptable messaging framework, empowering .NET developers to construct scalable and resilient distributed systems via a message-based architectural paradigm�just what we need.

### Why Opt for MassTransit?

Upon perusing the official documentation for MassTransit and gaining insight into how RabbitMQ and Azure Service Bus operate, several compelling reasons for embracing MassTransit come to light. Here are the top five motivations for choosing MassTransit:

1. **Streamlined Development**: MassTransit streamlines the programming model for crafting message-based applications, handling intricate details such as message serialization, routing, and transport. This liberates developers to concentrate on crafting business logic.

1. **Language and Platform Interoperability**: MassTransit accommodates a wide array of messaging transports and protocols, simplifying the task of building distributed systems that seamlessly communicate across diverse platforms and programming languages. This opens the door to utilizing the preferred languages and platforms of individual teams while ensuring seamless intercommunication.

1. **Resilience and Fault Tolerance**: MassTransit incorporates features like message retries and circuit breakers, facilitating the creation of highly resilient and fault-tolerant distributed systems. These features guarantee reliable message delivery even in the face of network disruptions and other challenges.

1. **Scalability**: MassTransit is engineered to support the scaling of distributed systems with ease. It offers features like load balancing and partitioning, enabling the handling of substantial message volumes and their distribution across multiple nodes.

1. **Open-Source and Community-Driven**: As an open-source project with an active community of contributors, MassTransit offers access to a wide range of features, bug fixes, and enhancements contributed by community members. This collaborative spirit enhances its utility.

## A quick introduction to RabbitMQ

Before we dive into the practical implementation, it's crucial to establish a foundational understanding of RabbitMQ, including the intricacies of queues and exchanges.

RabbitMQ serves as open-source message broker software, providing a messaging system for the exchange of messages between distinct applications and services. It stands as a robust and scalable messaging platform widely embraced in distributed systems.

RabbitMQ operates by implementing the Advanced Message Queuing Protocol (AMQP), a standardized messaging protocol for message-oriented middleware. This software supports various messaging patterns, including publish/subscribe, request/response, and work queues. It provides message queues, exchanges, and bindings, enabling different applications and services to communicate in a distributed environment.

RabbitMQ enjoys compatibility with a wide spectrum of programming languages and platforms, encompassing Java, .NET, Python, Ruby, and more. Furthermore, it supports various messaging protocols, including AMQP, STOMP, MQTT, and HTTP.

### Reasons to Leverage RabbitMQ

RabbitMQ emerges as a dependable and scalable messaging platform designed for distributed systems. Here are key features that make RabbitMQ a compelling choice:

1. **High Availability**: RabbitMQ supports clustering, fostering high availability and fault tolerance by connecting multiple nodes.

1. **Routing**: RabbitMQ facilitates diverse routing algorithms and message exchange types, ensuring messages reach their intended destinations accurately.

1. **Message Durability**: By persisting messages to disk, RabbitMQ ensures message durability, preventing message loss even in the event of system failures.

1. **Security**: RabbitMQ offers robust security measures, including SSL/TLS encryption, access control, and user authentication.

### RabbitMQ Queues

A RabbitMQ message queue functions as a buffer, temporarily storing messages until they are ready for processing by the consuming application or service. Each message within a queue possesses a unique identifier and remains in memory or on disk until it is either consumed by a consumer or reaches its predefined time-to-live (TTL). RabbitMQ queues can be configured to support various message delivery types, such as round-robin, priority-based, or topic-based routing.

At a high level, consider a message queue as a sizable buffer capable of retaining all your messages, awaiting consumption by the intended consumers.

Below is a diagram showing at a high level how a message queue works. You have one or more producers (the client) generating messages and publishing them to a queue. 
One or more consumers (the worker) can then consume the messages in the queue.

``` mermaid
sequenceDiagram
    participant Client
    participant Message Queue
    participant Worker

    Client->>Message Queue: Send Message
    Message Queue->>Worker: Enqueue Message
    Worker->>Message Queue: Dequeue Message
    Message Queue->>Worker: Return Message
    Worker->>Client: Process Message
```

### RabbitMQ Exchanges

With the basics of queues understood, let's delve further into exchanges. An exchange, a core component of a message broker, receives messages from producers and directs them to message queues based on specific routing rules. Exchanges are responsible for receiving messages and routing them to one or more queues, determined by message type and routing keys.

How does this work in practice? When a producer dispatches a message to RabbitMQ, it first reaches an exchange, which subsequently routes the message to one or more message queues based on message type and routing key. MassTransit, by default, employs the Fanout routing algorithm, which is the focus of this tutorial. However, for reference, a list of different exchange types supported by RabbitMQ concerning message routing is provided:

1. **Direct Exchange** - Direct exchanges route messages to a queue based on the exact match between the routing key and the binding key.
1. **Fanout Exchange** - Fanout exchanges route messages to all the queues that are bound to them, regardless of the routing key. (this is the one we will be working with in this tutorial)
1. **Topic Exchange** - Topic exchanges route messages based on pattern matching between the routing key and the binding key. The routing key can contain wildcards, allowing for more flexible routing.
1. **Headers Exchange** - Headers exchanges route messages based on the headers of the message, which can be any key-value pair.

The illustration below provides an overview of how the Fanout Exchange operates in RabbitMQ.

``` mermaid
graph TD
subgraph Producer
    A[Producer] -->|Message| B(Fanout Exchange)
end
subgraph Fanout Exchange
    B -->|Message| C(Queue 1)
    B -->|Message| D(Queue 2)
    B -->|Message| E(Queue 3)
end
subgraph Queue 3
    E -->|Message| H(Consumer 3)
end
subgraph Queue 2
    D -->|Message| G(Consumer 2)
end
subgraph Queue 1
    C -->|Message| F(Consumer 1)
end
```

## Implementation of MassTransit in .NET Core

Now, let's proceed to the part you've eagerly anticipated. If you've read through the introductions, my compliments. I acknowledge it was a substantial amount of information, but it is essential to possess this knowledge to construct a distributed application and grasp the underlying mechanisms.

### Deploy RabbitMQ using Docker

Our initial task is to ensure a running instance of RabbitMQ. I will employ Docker to launch and manage RabbitMQ. You can use PowerShell or your terminal to execute the following command, compatible with both Windows and Linux environments:

``` bash
docker run -d --hostname twc-rabbitmq-server --name rabbitmq -p 5672:5672 -p 15672:15672 rabbitmq:3.13-rc-management
```

What transpires when you execute this command?

- The most recent available management image of RabbitMQ from Docker is retrieved.
- The hostname is configured as "twc-rabbitmq-server," which you can customize as desired.
- The container is named "rabbitmq."
- Ports 5672 and 15672 are exposed for the container, mapping to the same ports internally. Port 5672 is the designated port for sending and publishing messages, while port 15672 serves as the administration panel accessible via a web browser.
- 
With Docker's assistance, RabbitMQ is up and running. To explore the RabbitMQ administration panel, open your web browser and navigate to "localhost:15672." You can log in using the default credentials: "guest/guest." We will delve into the utilization of this web panel later in this article.
You may want to quickly check the logs to ensure no errors occurred. You can do so by executing the following command:

``` bash
docker logs rabbitmq
```



[mass]: https://masstransit.io
[docker]: https://docs.docker.com/get-docker
[rabbit]: https://www.rabbitmq.com
[exchange]: https://www.rabbitmq.com/tutorials/amqp-concepts.html
[rd]: https://hub.docker.com/_/rabbitmq