# Mastering the Art of API Design

> Source: https://blog.bytebytego.com/p/mastering-the-art-of-api-design
> Collected: 2026-04-19
> Published: 2023-05-10

In this three-part series, we talk about API design:

- Why is the "API First" principle important for an organization?
- How do we design effective and safe APIs?
- What are the tools that can boost productivity?

APIs are a set of protocols that define how system components interact with each other. As architectural styles evolve, APIs have gained prominence in recent years. The diagram below shows how the rise of microservices and cloud-native applications brings further granularity to services. In-process calls in monolithic applications transition to inter-process calls in microservice and serverless applications. Additionally, each process might reside on a different physical server, and service calls can fail due to various network issues.

Increased service complexity emphasizes the need for more disciplined API designs.

## API First

Over the past decade, "API First" has emerged as a popular software development model. It prioritizes API design before system design. Various functional teams and systems use APIs as a shared communication language. For example, frontend developers, backend developers, and QA teams work together to design APIs based on system requirements. These APIs serve as specifications for business requirements and system designs. Each team then works independently, and they reconvene during the dev testing phase.

The diagram below compares the "Code First" and "API First" approaches. In the "Code First" model, APIs are byproducts of system designs, often referred to as "documentation". The "API First" model begins with API specifications and concludes with API-driven tests, making APIs the driving force behind the entire software development cycle.

"API First" offers several advantages:

- Improved system integration. "API First" encourages developers to carefully consider system interactions from the project's outset, reducing the need for ongoing modifications during development.
- Enhanced collaboration and quality. APIs serve as a shared specification within the organization, allowing developers, testers, and DevOps to work independently. Agreeing on APIs at the project's beginning helps eliminate uncertainties and boost software quality.
- Increased scalability. With defined interfaces for each service, scaling becomes more manageable by spinning up new instances and adjusting load balancer settings.

In addition to efficiency and transparency, the API-first design also fosters network effects.

In 2002, Jeff Bezos issued the famous API mandate, an early version of "API First". As a result, systems within the organization became Lego-like building blocks, creating an open ecosystem. The value of this ecosystem grows as more participants leverage APIs to develop new products or services, leading to network effects. Amazon Web Services (AWS), for example, has since become a significant revenue source for the company.

It is quite visionary to mandate that all systems be designed with scalability and flexibility in mind. As a result, the company can adapt swiftly to changing business conditions.

## API Architectural Styles

Different API architectural styles use different communication protocols and data formats.

### 1. REST

Introduced in 2000 by Roy Fielding, REST (Representational State Transfer) is the most widely used style between front-end clients and back-end services. In a RESTful architecture, every component is a resource, accessed using standard HTTP methods like GET, POST, PUT, and DELETE. Payload formats can be JSON, XML, HTML, or plain text.

REST defines six architectural constraints which make a web service truly RESTful:

- Uniform interface. We must define API interfaces for resources.
- Client–Server. Client-side application and server-side applications must evolve separately.
- Stateless. All client-server interactions are stateless. The server treats every request as new.
- Cacheable. Data and responses are cached wherever possible.
- Layered system. APIs, services, and data can be deployed on different servers.
- Code on demand (optional). This optional constraint allows the server to return executable code if needed.

### 2. GraphQL

GraphQL was proposed in 2015 by Meta. It provides a schema and type system suitable for complex systems with graph-like relationships between entities. It handles complex queries with nested data structures. For example, GraphQL can retrieve user and order information in one call, while REST requires multiple calls to different endpoints.

Note that GraphQL is not a replacement for REST. It can be built upon existing REST services, making migration less invasive.

However, GraphQL brings complexities. For example, GraphQL can expose more resource fields than necessary if the queries are not carefully designed. In addition, caching is more challenging due to increased query flexibility.

Organizations should evaluate the need for GraphQL. It has a steeper learning curve for understanding the new query language and new schema design.

### 3. WebSocket

WebSocket is a protocol that provides full-duplex communications over TCP. Clients establish WebSockets to receive real-time updates from the back-end services. Unlike REST, which always "pulls" data, WebSocket enables data to be "pushed". Applications, like online gaming, stock trading, and messaging apps leverage WebSocket for real-time communication.

### 4. Webhook

Webhooks are commonly used for third-party asynchronous API calls. With a webhook, one application can register to receive updates from another.

As SaaS (Software-as-a-Service) becomes popular, webhook use has become widespread for integrating SaaS services. Many SaaS services include webhook support in their APIs.

For example, we use Stripe or Paypal for payment channels and register a webhook for payment results. When a third-party payment service completes its process, it notifies our payment service whether the payment was successful or failed. Webhook calls typically form part of the system's state machine.

### 5. gRPC

Released in 2016 by Google, gRPC is a modern, open-sourced RPC (Remote Procedure Call) framework used for server-to-server communication in distributed systems. gRPC provides language-agnostic APIs and uses Protocol Buffers for data serialization in communications.

Compared with REST, gRPC offers code-generation tools that help generate client and server stubs, reducing coding effort for data transmission. gRPC is based on HTTP/2, allowing multiplexing and streaming, so clients and servers can send and receive data simultaneously.

A drawback of RPC frameworks is that they make remote procedure calls look like local procedure calls, masking the complexity of handling unreliable networks. This can lead to serious bugs if developers don't realize that responses may be dropped intermittently due to network issues.

### 6. SOAP

SOAP (Simple Object Access Protocol) uses XML payloads for communication between internal systems.

### 7. Kafka

Kafka is often used as a messaging layer to facilitate publish-subscribe-based communications. This event-streaming paradigm differs from the request-response paradigm.

The order service sends orders to the payment service via Kafka because the payment service is usually slower than the order service. Using REST APIs for payments may lead to long waits for responses, affecting processing throughput.

Additionally, Kafka supports fan-out, which is an electronic term for sending one input to multiple outputs. If the request-response paradigm were used for fan-out, the codebase would quickly become unmaintainable.

## Popularity

Postman published a report presenting the results of a survey on the popularity of each architectural style. REST remains the dominant style, with 89% of survey participants choosing it. Webhooks (35%), GraphQL (28%), and gRPC (11%) have gained popularity compared to the year before. "Their growth in popularity comes as gRPC is used for internal microservices and GraphQL for stitching together disparate data sources."

## Comparison

Each style was developed for a specific purpose. For example, GraphQL gained increased popularity in large firms because it streamlines relationships among complex REST-based APIs; RPC became the standard protocol for microservices as firms adopted microservice architecture more widely.

When designing a system, we need to choose appropriate interaction methods for different scenarios. In some extreme situations, none of these architectural styles are suitable, and we must develop proprietary communication protocols. This requirement is typical in low-latency trading applications where RPC is too slow.
