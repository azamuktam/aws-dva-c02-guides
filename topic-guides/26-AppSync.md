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
<empty-block/>
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
<table header-row="true">
<tr>
<td>Requirement</td>
<td>Best fit</td>
</tr>
<tr>
<td>**GraphQL API**</td>
<td>**AppSync**</td>
</tr>
<tr>
<td>**Client chooses exactly which fields to receive**</td>
<td>**AppSync**</td>
</tr>
<tr>
<td>**Multiple data sources behind one API**</td>
<td>**AppSync**</td>
</tr>
<tr>
<td>**GraphQL real-time subscriptions**</td>
<td>**AppSync**</td>
</tr>
<tr>
<td>**Mobile offline synchronization**</td>
<td>**AppSync**</td>
</tr>
<tr>
<td>REST API</td>
<td>**API Gateway**</td>
</tr>
<tr>
<td>HTTP API</td>
<td>**API Gateway**</td>
</tr>
<tr>
<td>Custom WebSocket API</td>
<td>**API Gateway**</td>
</tr>
<tr>
<td>API Gateway stages / REST-style API management</td>
<td>**API Gateway**</td>
</tr>
</table>
Don't think **"AppSync = replacement for API Gateway."** They solve different API patterns. AWS supports using both when appropriate. 
**Exam clues**
<table header-row="true">
<tr>
<td>Question says...</td>
<td>Think...</td>
</tr>
<tr>
<td>**GraphQL**</td>
<td>**AppSync**</td>
</tr>
<tr>
<td>**Client requests only the fields it needs**</td>
<td>**AppSync**</td>
</tr>
<tr>
<td>**Multiple data sources through one API**</td>
<td>**AppSync**</td>
</tr>
<tr>
<td>**Real-time GraphQL updates**</td>
<td>**AppSync subscriptions**</td>
</tr>
<tr>
<td>**Mobile app + offline synchronization**</td>
<td>**AppSync**</td>
</tr>
<tr>
<td>**Chat / collaborative application**</td>
<td>**AppSync**</td>
</tr>
<tr>
<td>REST API</td>
<td>**API Gateway**</td>
</tr>
<tr>
<td>HTTP API</td>
<td>**API Gateway**</td>
</tr>
<tr>
<td>Custom WebSocket routes/connections</td>
<td>**API Gateway WebSocket**</td>
</tr>
</table>
### One-line memory
> **AppSync = GraphQL + multiple data sources + real-time + mobile/offline synchronization.** 
**Exam memory:**
> **Offline + synchronize across devices → AppSync**
> **Real-time + collaborative → AppSync**
> **GraphQL → AppSync**
