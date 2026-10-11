## AWS AppSync

**Mental model**

> **AppSync = managed GraphQL API + real-time data + multi-source data**

It lets mobile/web clients communicate with multiple backend data sources through a **single GraphQL API**. AWS also provides real-time capabilities through subscriptions and AppSync Events.

```plain text
Mobile / Web
     │
     ▼
  AppSync
     │
     ├── DynamoDB
     ├── Lambda
     ├── Aurora Serverless
     ├── OpenSearch
     └── HTTP APIs
```

**Why AppSync exists**

A mobile app may need data from several places.

Without GraphQL:

```plain text
Mobile
 ├──→ /users
 ├──→ /orders
 └──→ /products
```

With AppSync:

```plain text
Mobile
   │
   ▼
AppSync
   ├──→ Users
   ├──→ Orders
   └──→ Products
```

The client asks for **exactly the fields it needs** in one GraphQL request. This is especially useful when clients need different combinations of data or when data comes from multiple sources.

**Main features**

**Offline synchronization**

- Mobile apps can **work offline**
- Changes can be **stored locally**
- When connectivity returns, changes can **synchronize with the backend**
- Data can be synchronized across **devices/users**
- AppSync supports **conflict resolution** for concurrent changes
- Common for **collaborative apps, chat, group study, and offline-first mobile apps**

**GraphQL API**

Clients use:

- **Queries** → read data
- **Mutations** → create/update/delete data
- **Subscriptions** → receive real-time updates

```graphql
{
  user {
    name
    orders {
      id
      total
    }
  }
}
```

GraphQL lets the client specify the data it wants instead of receiving a fixed REST response.

**Multiple data sources**

AppSync can connect one GraphQL API to:

- DynamoDB
- Lambda
- Aurora Serverless
- OpenSearch
- HTTP endpoints
- Other supported sources

This makes AppSync useful as a **single API layer over multiple backend services**.

**Real-time**

GraphQL **subscriptions** allow clients to receive changes in real time through managed WebSocket connections.

Use it for:

- Chat
- Live dashboards
- Collaborative applications
- Notifications
- Live status updates

**Offline synchronization**

For mobile applications, AppSync can be used with AWS client-side tooling for **offline access and synchronization**.

Typical exam clue:

> **Mobile app + offline changes + synchronize when online → AppSync**

This is exactly the pattern in your university/student-portal question.

**Resolvers**

A **resolver** connects a GraphQL field to a data source.

```plain text
GraphQL field
     ↓
 Resolver
     ↓
DynamoDB / Lambda / HTTP / etc.
```

Resolvers can use **VTL or JavaScript**.

**Authorization**

AppSync supports several authorization methods, including:

- API key
- IAM
- Cognito User Pools
- OIDC
- Lambda authorization

Choose based on the application's authentication requirements.

**AppSync vs API Gateway**

| Requirement | Best fit |
| --- | --- |
| **GraphQL API** | **AppSync** |
| **Client chooses exactly which fields to receive** | **AppSync** |
| **Multiple data sources behind one API** | **AppSync** |
| **GraphQL real-time subscriptions** | **AppSync** |
| **Mobile offline synchronization** | **AppSync** |
| REST API | **API Gateway** |
| HTTP API | **API Gateway** |
| Custom WebSocket API | **API Gateway** |
| API Gateway stages / REST-style API management | **API Gateway** |

Don't think **"AppSync = replacement for API Gateway."** They solve different API patterns. AWS supports using both when appropriate.

**Exam clues**

| Question says... | Think... |
| --- | --- |
| **GraphQL** | **AppSync** |
| **Client requests only the fields it needs** | **AppSync** |
| **Multiple data sources through one API** | **AppSync** |
| **Real-time GraphQL updates** | **AppSync subscriptions** |
| **Mobile app + offline synchronization** | **AppSync** |
| **Chat / collaborative application** | **AppSync** |
| REST API | **API Gateway** |
| HTTP API | **API Gateway** |
| Custom WebSocket routes/connections | **API Gateway WebSocket** |

### One-line memory

> **AppSync = GraphQL + multiple data sources + real-time + mobile/offline synchronization.**

**Exam memory:**

> **Offline + synchronize across devices → AppSync**
> **Real-time + collaborative → AppSync**
> **GraphQL → AppSync**

# Question patterns

> A mobile app needs a single flexible GraphQL API over several backend data sources. Which service?
> → **AWS AppSync**

> Clients must receive real-time updates when data changes. Which AppSync feature?
> → **subscriptions (managed WebSockets)**

> A mobile app must keep working offline and synchronize changes when connectivity returns. Which service?
> → **AWS AppSync offline synchronization**

> What does a resolver connect?
> → **a GraphQL field to a data source**

> What languages can AppSync resolvers use?
> → **VTL (Velocity Template Language) or JavaScript**

> Which authorization methods does AppSync support?
> → **API key, IAM, Cognito User Pools, OIDC, and Lambda authorization**

> Which GraphQL operation reads data?
> → **query**

> Which GraphQL operation creates, updates or deletes data?
> → **mutation**

> Which GraphQL operation pushes real-time updates to clients?
> → **subscription**

> A client must fetch exactly the fields it needs in one request instead of a fixed response. Which service?
> → **AppSync (GraphQL)**

> Data must be combined from DynamoDB, Lambda and an HTTP API behind one endpoint. Which service?
> → **AppSync (multiple data sources)**

> Name data sources AppSync can connect to.
> → **DynamoDB, Lambda, Aurora Serverless, OpenSearch, HTTP endpoints and other supported sources**

> Two users edit the same record concurrently and changes must be merged. Which AppSync capability?
> → **conflict resolution**

> A collaborative chat or group-study app needs live updates across devices. Which service and feature?
> → **AppSync (real-time subscriptions)**

> Is AppSync a drop-in replacement for API Gateway?
> → **No — they solve different API patterns and can be used together**

> A REST API must be exposed. Which service?
> → **API Gateway**

> An HTTP API must be exposed. Which service?
> → **API Gateway**

> A custom WebSocket API with your own routes/connections is needed. Which service?
> → **API Gateway WebSocket**

> API stages and REST-style API management are needed. Which service?
> → **API Gateway**

> A single API layer must sit over multiple backend services for mobile/web clients. Which service?
> → **AppSync**

> What does AppSync use to push real-time changes to clients at the transport level?
> → **managed WebSocket connections**

> Which clue phrase maps to AppSync: "offline changes, synchronize when online"?
> → **AppSync**

> Which clue phrase maps to AppSync: "client requests only the fields it needs"?
> → **AppSync**

> Which service provides a managed GraphQL API with real-time and multi-source data?
> → **AWS AppSync**

> AppSync is used for collaborative/chat apps, live dashboards and notifications. What enables the live updates?
> → **GraphQL subscriptions**

# Pocket card

| Keyword | Answer |
| --- | --- |
| GraphQL | **AppSync** |
| Real-time | **subscriptions (WebSockets)** |
| Resolve logic | **VTL or JavaScript resolvers** |
| Auth | **API key / IAM / Cognito / OIDC / Lambda** |
| Read data | **query** |
| Write data | **mutation** |
| Live updates | **subscription** |
| Client picks fields | **AppSync (GraphQL)** |
| Multiple data sources | **AppSync** |
| Offline + sync across devices | **AppSync** |
| Collaborative / chat | **AppSync** |
| Concurrent edits | **conflict resolution** |
| Resolver connects | **field → data source** |
| Data sources | **DynamoDB, Lambda, Aurora Serverless, OpenSearch, HTTP** |
| REST API | **API Gateway** |
| HTTP API | **API Gateway** |
| Custom WebSocket | **API Gateway WebSocket** |
| API stages / REST mgmt | **API Gateway** |
| Replacement for API Gateway | **No — different patterns** |
| Single API over many services | **AppSync** |
| Live dashboards / notifications | **AppSync subscriptions** |
| Mental model | **managed GraphQL + real-time + multi-source** |
