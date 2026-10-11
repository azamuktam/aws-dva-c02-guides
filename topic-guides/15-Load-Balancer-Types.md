# PART 3 — CONNECTING THINGS: APIs, EVENTS, AND STREAMS
Two distinct problems live here.
**Exposing an interface** — API Gateway, ALB, CloudFront, Route 53. Something outside needs to reach your code.
**Moving work between components** — SQS, SNS, EventBridge, Step Functions, Kinesis. Components need to hand work to each other without being tightly coupled.
The second group is where most people lose points, because five services overlap and the exam picks between them on subtle grounds. The organising question is: **does order matter, does the data need to be replayable, and how many consumers need the same message?**
<table header-row="true">
<tr>
<td></td>
<td>Consumers per message</td>
<td>Ordering</td>
<td>Replay</td>
<td>Built for</td>
</tr>
<tr>
<td>**SQS**</td>
<td>One</td>
<td>FIFO queues only</td>
<td>❌</td>
<td>Decoupling, load levelling, buffering</td>
</tr>
<tr>
<td>**SNS**</td>
<td>Many (fan-out)</td>
<td>FIFO topics only</td>
<td>❌</td>
<td>Push notification, fan-out</td>
</tr>
<tr>
<td>**EventBridge**</td>
<td>Many (routed by rule)</td>
<td>❌</td>
<td>✅ **Archive + replay**</td>
<td>Event routing on content</td>
</tr>
<tr>
<td>**Kinesis Data Streams**</td>
<td>Many, independently, same data</td>
<td>✅ Per partition key</td>
<td>✅ Up to 365 days</td>
<td>Ordered streaming, analytics</td>
</tr>
<tr>
<td>**Step Functions**</td>
<td>n/a</td>
<td>Sequential by definition</td>
<td>n/a</td>
<td>Orchestrating a multi-step process</td>
</tr>
</table>
---

# AWS Load Balancer Types
<table header-row="true">
<tr>
<td>Load Balancer</td>
<td>Layer</td>
<td>Use Case</td>
</tr>
<tr>
<td>**Application Load Balancer (ALB)**</td>
<td>Layer 7 (HTTP/HTTPS)</td>
<td>Web applications, path-based and host-based routing</td>
</tr>
<tr>
<td>**Network Load Balancer (NLB)**</td>
<td>Layer 4 (TCP/UDP/TLS)</td>
<td>High performance, low latency, static IP support</td>
</tr>
<tr>
<td>**Gateway Load Balancer (GWLB)**</td>
<td>Layer 3</td>
<td>Deploy security appliances like firewalls and IDS</td>
</tr>
<tr>
<td>**Classic Load Balancer (CLB)**</td>
<td>Layer 4 / Layer 7</td>
<td>Old generation load balancer (legacy)</td>
</tr>
</table>
**Exam memory**
- **ALB → HTTP/HTTPS/Websocket → Layer 7**
- **NLB → TCP/UDP → Layer 4 → Static IP**
- **GWLB → Security appliances**
- **CLB → Old ELB**
AWS Elastic Load Balancing supports these four load balancer types.
