# A Crash Course on REST APIs

> Source: https://blog.bytebytego.com/p/a-crash-course-on-rest-apis
> Collected: 2026-04-29
> Published: 2024-05-30
> Author: ByteByteGo

Application Programming Interfaces (APIs) are the backbone of software communication.

APIs have been around for a long time in one form or the other:

- In the 60s and 70s, we had subroutines and libraries to share code and functionality between programs.
- In the 1980s, Remote Procedure Calls (RPC) emerged, allowing programs running on different computers to execute procedures on each other.
- With the widespread adoption of the Internet in the 2000s, web services such as SOAP became widely adopted.
- The late 2000s and early 2010s marked the rise of RESTful APIs, which have since become the dominant approach due to their simplicity and scalability.

## Introduction to REST APIs

REST stands for Representational State Transfer. Roy Fielding coined the term in his doctoral dissertation in 2000. He defined REST as an architectural style for designing networked applications.

### Key Terminologies

**HTTP (Hypertext Transfer Protocol)** - the foundation protocol for communication on the web. Stateless protocol — each request is independent, the server doesn't keep any information about previous requests. Follows a request-response model. Each HTTP request consists of a method (GET, POST, PUT, DELETE), headers, and an optional body. HTTP responses include a status code, headers, and an optional body.

**URLs (Uniform Resource Locators)** - addresses used to reach resources on the web. A typical URL consists of the protocol (HTTP or HTTPS), the domain name, the path to the resource (such as /api/users), and optional query parameters.

**Client-Server Architecture** - the client sends requests to the server. The server processes the requests and sends back responses. The client takes care of presenting the user interface, while the server handles the business logic, data processing, and storage. They communicate over a network typically using HTTP.

## Resource-Based Architecture

In REST, the core concept is the resource. A resource is any piece of information that can be named and accessed through a URL. It can be a user, a product, an order, or even a collection of other resources. Resources are represented using standard formats such as JSON or XML.

### Resource Naming Conventions

- **Use nouns, not verbs** — Resource URLs should represent entities or concepts, not actions. Use `/users` instead of `/getUsers` or `/createUser`.
- **Use plural nouns for collections** — For a collection of resources, use plural nouns. For example, `/products` represents a collection of product resources.
- **Use hierarchical paths** — The URL structure should reflect resources with a hierarchical or nested relationship. For example, `/orders/123/items` represent the collection of items within a specific order.
- **Use hyphens to separate words** — If a resource name consists of multiple words, use hyphens. For example, `/product-categories` or `/user-profiles`.
- **Use lowercase letters** — Resource names should be lowercase to maintain consistency and avoid confusion.

## HTTP Methods with REST APIs

HTTP methods are used in REST APIs to perform different operations on resources. Together, these HTTP methods form the foundation of CRUD operations.

**GET** - Retrieves or reads a resource from the server. Safe and idempotent — multiple identical requests have the same effect as a single request and should not modify the resource. For example, `GET /users` retrieves a list of users, `GET /users/456` retrieves a specific user with ID 456.

**POST** - Creates a new resource on the server. Data is sent in the request body using JSON or XML format. For example, `POST /users` creates a new user based on the data provided in the request body.

**PUT** - Updates or replaces an existing resource on the server. Idempotent. The client sends the complete representation of the updated resource in the request body. For example, `PUT /users/123` updates the user with ID 123 with the data provided in the request body.

**PATCH** - Partially updates an existing resource on the server. Unlike PUT, which requires sending the complete representation, PATCH allows sending only the changes to be applied. For example, `PATCH /users/123` partially updates the user with ID 123 based on the changes provided in the request body.

**DELETE** - Deletes a resource from the server. For example, `DELETE /users/123` deletes the user with ID 123.

## API Design Best Practices

### API Versioning

There are several API versioning strategies to choose from.

**URL Versioning** — the version number is included as part of the API's URL. Example: `/v1/users`. Simple and explicit, but can lead to long URLs. The URL has to be updated in the client code when switching versions.

**Query Parameter Versioning** — the version number is passed as a query parameter. Example: `GET /users?version=1`. Keeps the base URL clean and allows easy switching between versions. However, query parameters are typically used for filtering/sorting — using one for versioning feels less intuitive and might be overlooked by developers.

**Custom Header Versioning** — a custom header is used to specify the API version. Example: `X-API-Version: 1`. Same advantages as query parameter versioning, but requires additional header configuration in the client code.

### Pagination

Pagination limits the number of results returned in a single API response, especially when dealing with large datasets. Allows clients to retrieve data in smaller, more manageable chunks.

Common pagination parameters: `page` (current page number) and `limit` (number of items per page). The API should return pagination metadata in the response: total number of items, total pages, and links to next and previous pages.

Example: `GET /users?page=2&limit=10` retrieves the second page of users with 10 items per page.

### Filtering

Filtering lets clients narrow down the number of records based on specific criteria, reducing the amount of data transferred over the network. Implemented using query parameters where the client specifies the field and value to filter on.

Example: `GET /products?category=electronics&price_max=100` retrieves products in the electronics category with a price less than or equal to 100.

### Sorting

Sorting lets clients specify the order in which the results should be returned. Implemented using query parameters where the client specifies the field to sort on and the sorting order (ascending or descending).

Example: `GET /users?sort=name&order=asc` retrieves users sorted by name in ascending order.

### Error Handling

The API should return meaningful error messages and appropriate HTTP status codes to indicate the type of error. Error responses should include a clear error message, an error code or type, and additional details if necessary. Consistent error handling across API endpoints helps clients handle and display errors effectively.

### Documentation

Comprehensive and up-to-date documentation is a key pillar of API accessibility. The documentation should include details about endpoints, request/response formats, authentication, error handling, and code examples.

Tools like **Swagger** or **OpenAPI** can be used to generate interactive documentation from API specifications. Many frameworks provide code-based support for Swagger, automatically updating the documentation as the developer modifies the code.

## Authentication and Authorization

### Authentication

Authentication is the process of verifying the identity of the user or client accessing the API. It answers: "Who are you?"

Common authentication mechanisms:
- Username and password-based authentication
- API keys or tokens (e.g., JWTs)
- OAuth for delegated access

Authentication is important for: protecting sensitive data and functionality, tracking and auditing user actions, personalization and customization based on identity.

### Authorization

Authorization is the process of determining what actions or resources a user or client is allowed to access once their identity is authenticated. It answers: "What are you allowed to do?"

Common authorization mechanisms:
- **Role-Based Access Control (RBAC)** — Users are assigned roles, and permissions are granted based on those roles.
- **Attribute-Based Access Control (ABAC)** — Access is granted based on attributes or characteristics of the user, resource, or environment.
- **OAuth scopes** — OAuth tokens include scopes that define the permissions and access levels granted to the client.

Authorization is important for: ensuring users can only access resources they are authorized to, protecting sensitive data from unauthorized access or modification, implementing granular access control based on user roles, permissions, or attributes.

## Scalability and Performance

### Stateless Architecture

Design REST APIs to be stateless — each request from the client should contain all the necessary information for the server to process it.

Practical implications:
- Not storing session data on the server side (hinders scalability and makes distributing requests across multiple servers difficult).
- Using stateless authentication mechanisms like JWTs or API keys.
- Alternatively, when using sessions, store session information in a separate storage rather than on the API server instance.

### Horizontal Scaling

Building REST APIs using stateless architecture makes them horizontally scalable — add more servers to handle increased traffic and load. Incoming requests are distributed between servers using load balancers.

### Caching

Implement caching mechanisms to reduce the load on the API server and improve response times.

- **HTTP caching headers** (`Cache-Control`, `ETag`) — enable client-side caching and avoid unnecessary requests for unchanged resources.
- **Server-side caching** — use a distributed cache like Redis or a CDN to store the results of expensive computations or frequently accessed data.
- **Cache-Aside pattern** — application checks the cache first; on a miss, loads from the database and populates the cache.

### Efficient Data Serialization

Choose efficient data serialization formats like JSON or Protocol Buffers to minimize the size of the payload transferred over the network. Avoid sending unnecessary data in the response payload.

### Asynchronous Processing and Message Queues

**Asynchronous Processing** — tasks are executed independently of the main program flow. Instead of blocking the API request until the task is completed, the API initiates the task asynchronously and immediately returns a response to the client indicating that the request has been accepted and is being processed. The client can then poll the API or receive notifications (via webhooks) to check the status of the task.

**Message Queues** — producers (APIs) send messages to a queue while consumers (background workers) retrieve and process those messages independently. The API can quickly enqueue a message representing a task and continue processing other requests while consumers process messages asynchronously.

Example flow for a resource-intensive task:
1. Client sends POST request to initiate the task (e.g., generating a complex report).
2. API validates input, enqueues a message representing the task, returns 202 Accepted with a task identifier.
3. Background consumers retrieve messages from the queue and process them asynchronously.
4. Result is stored in a database or storage system.
5. Client polls using the task identifier or receives a webhook notification when complete.

### Monitoring and Logging

Implement comprehensive monitoring and logging mechanisms to track the performance and health of REST APIs.

Key metrics to track:
- Response times
- Error rates
- Resource utilization

Use a centralized logging solution to collect and analyze log data from multiple servers.
