# PART 3 — CONNECTING THINGS: APIs, EVENTS, AND STREAMS
Two distinct problems live here.
**Exposing an interface** — API Gateway, ALB, CloudFront, Route 53. Something outside needs to reach your code.
**Moving work between components** — SQS, SNS, EventBridge, Step Functions, Kinesis. Components need to hand work to each other without being tightly coupled.
The second group is where most people lose points, because five services overlap and the exam picks between them on subtle grounds. The organising question is: **does order matter, does the data need to be replayable, and how many consumers need the same message?**
|    | Consumers per message | Ordering | Replay | Built for |
| --- | --- | --- | --- | --- |
| **SQS** | One | FIFO queues only | ❌ | Decoupling, load levelling, buffering |
| **SNS** | Many (fan-out) | FIFO topics only | ❌ | Push notification, fan-out |
| **EventBridge** | Many (routed by rule) | ❌ | ✅ **Archive + replay** | Event routing on content |
| **Kinesis Data Streams** | Many, independently, same data | ✅ Per partition key | ✅ Up to 365 days | Ordered streaming, analytics |
| **Step Functions** | n/a | Sequential by definition | n/a | Orchestrating a multi-step process |
---

# AWS Load Balancer Types
| Load Balancer | Layer | Use Case |
| --- | --- | --- |
| **Application Load Balancer (ALB)** | Layer 7 (HTTP/HTTPS) | Web applications, path-based and host-based routing |
| **Network Load Balancer (NLB)** | Layer 4 (TCP/UDP/TLS) | High performance, low latency, static IP support |
| **Gateway Load Balancer (GWLB)** | Layer 3 | Deploy security appliances like firewalls and IDS |
| **Classic Load Balancer (CLB)** | Layer 4 / Layer 7 | Old generation load balancer (legacy) |
**Exam memory**
- **ALB → HTTP/HTTPS/Websocket → Layer 7**
- **NLB → TCP/UDP → Layer 4 → Static IP**
- **GWLB → Security appliances**
- **CLB → Old ELB**
AWS Elastic Load Balancing supports these four load balancer types.
