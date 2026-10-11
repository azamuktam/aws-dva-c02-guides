## VPC, Security Groups, and NACLs
Networking is in scope, but **designing** networking is explicitly out of scope for the target candidate. What you need is enough to connect your application and to diagnose “it can’t reach X”.
**The structure**
```plain text
Internet
   ↓
Internet Gateway
   ↓
Public Subnet   (route 0.0.0.0/0 → IGW)
   ├── Load Balancer
   └── NAT Gateway
   ↓
Private Subnet  (route 0.0.0.0/0 → NAT Gateway)
   ├── Application servers / Lambda ENIs / ECS tasks
   └── Database (RDS)
```
**VPC CIDR block** — the IP range, e.g. `10.0.0.0/16`. Must not overlap with networks you connect to.
**Subnet** — a range within the VPC, living in exactly **one Availability Zone**. A VPC spans AZs; a subnet does not. Use subnets in multiple AZs for high availability.
- **Public subnet** — has a route to an **Internet Gateway**. Resources need a public IP to be reachable.
- **Private subnet** — no direct internet route. Where application servers and databases belong.
```plain text
Public subnet
→ route table → Internet Gateway → Internet

Private subnet
→ no direct route to Internet Gateway
```
A private subnet **can still access the Internet** through a **NAT Gateway**:
```plain text
Private EC2 → NAT Gateway → Internet Gateway → Internet
```
But the Internet **cannot directly initiate a connection to the private EC2** through that NAT path.
So for your DVA exam:
> **Public vs private = primarily determined by the subnet's route to an Internet Gateway.**
**Internet Gateway (IGW)** — attaches to the VPC and enables two-way internet communication. Three things are required for a resource to reach the internet through it: the IGW attached, a route `0.0.0.0/0 → IGW`, and a public IP on the resource.
**NAT Gateway** — lets private-subnet resources make **outbound** connections to the internet (software updates, external APIs) while **blocking all inbound** connections. Lives in a *public* subnet.
**Route tables** — decide where traffic goes. Every subnet is associated with one.
```plain text
VPC
│
├── Subnet A
│    ├── EC2
│    └── Route Table A
│
└── Subnet B
     ├── EC2
     └── Route Table B
```
A **route table is associated with a subnet**. It controls where traffic from resources in that subnet should go.
### Example: public subnet
```plain text
VPC
│
└── Public Subnet
     │
     ├── EC2
     │
     └── Route Table
          │
          ├── 10.0.0.0/16 → local
          │
          └── 0.0.0.0/0 → Internet Gateway
```
If the EC2 wants to access Google:
```plain text
EC2
 ↓
Route Table
 ↓
"Google is not inside my VPC"
 ↓
0.0.0.0/0 → Internet Gateway
 ↓
Internet
```
If EC2 wants to talk to another EC2 **inside the same VPC**:
```plain text
EC2 A
 ↓
Route Table
 ↓
10.0.0.0/16 → local
 ↓
EC2 B
```
### The key relationship
> **VPC** = whole network
  **Subnet** = section of the VPC
  **Route table** = directions for traffic from that subnet
  **Internet Gateway** = door to the Internet
**ENI (Elastic Network Interface)** — a virtual network card carrying a private IP, optionally a public IP, security groups, and a MAC address. Used by EC2 and by **Lambda when attached to a VPC** — created automatically in that case.
**VPC Peering** — a private connection between two VPCs. **No transitive routing**: if A peers with B and B peers with C, A still cannot reach C.
**VPC Endpoints** — private access to AWS services without traversing the internet.
- **Gateway endpoint** — **S3 and DynamoDB only**. Free. Works via a route table entry.
- **Interface endpoint** — most other services, using **PrivateLink** and an ENI in your subnet. Charged hourly.
The practical developer relevance: a Lambda in a private subnet needing S3 can use a **gateway endpoint** instead of a NAT gateway, which is both cheaper and more secure.
**Security Groups vs NACLs — stateful versus stateless**
This distinction produces one of the most reliable troubleshooting questions on the exam.
**Security Group** — attached to an **instance/ENI**. **Stateful**: if you allow traffic in, the response is automatically allowed out. Allow rules **only** — you cannot express a deny. Default: all inbound denied, all outbound allowed. All rules are evaluated together.
**Network ACL** — attached to a **subnet**. **Stateless**: it has no memory of connections, so you must allow the inbound traffic **and** the outbound response explicitly. Supports **both allow and deny** rules, evaluated by **rule number, lowest first**, stopping at the first match.
| Feature | **Security Group** | **Network ACL** |
| --- | --- | --- |
| Level | Instance / ENI | **Subnet** |
| Stateful | ✅ Yes | ❌ **No** |
| Rules | **Allow only** | Allow **and Deny** |
| Return traffic | Automatic | **Must be allowed manually** |
| Rule order | All evaluated together | **Lowest number first**, first match wins |
| Main use | Primary instance protection | Additional subnet-level layer, blocking specific IPs |
The two diagnostic consequences:
- **You need to block a specific IP address** → only a **NACL** can do it, because security groups have no deny.
- **Inbound works but responses never arrive** → the NACL’s **outbound** rules are blocking the return traffic. A security group would have handled it automatically. This symptom points at a NACL every time.
**VPC Flow Logs**
Records **metadata** about IP traffic — not packet contents.
Captured: source IP, destination IP, ports, protocol, **ACCEPT/REJECT**, byte and packet counts.<br>Not captured: payloads, application data.
```plain text
action=REJECT  srcaddr=10.0.1.5  dstport=443
```
Can be enabled at the **VPC, subnet, or ENI** level, and delivered to CloudWatch Logs, S3, or Kinesis Data Firehose.
This is the tool that answers “why is this connection being blocked?” — a REJECT tells you a security group or NACL denied it, and the ports involved tell you which rule to look at.
**The observability comparison**
| Tool | Records |
| --- | --- |
| **VPC Flow Logs** | Network traffic metadata (accepted/rejected) |
| **CloudTrail** | **AWS API calls** — who did what |
| **CloudWatch Logs** | Application and service logs |
| **AWS Config** | Resource configuration state and compliance |
**AWS Config** tracks the **configuration and compliance state of AWS resources**, while **CloudTrail** records **who did what and when through API calls**.
**AWS Direct Connect** provides a dedicated network connection between** your on-premises network and AWS**, bypassing the public internet
**Recall check — VPC and network security**
| Component | Simple purpose |
| --- | --- |
| **Internet Gateway (IGW)** | Public resources ↔ Internet |
| **NAT Gateway** | Private resources → Internet |
| **Egress-only IGW** | Private **IPv6** resources → Internet |
| **VPC Endpoint** | Private resource → AWS service **without Internet (Gateway Endpoint, Interface endpoint)** |
| **VPC Peering** | Private VPC ↔ Private VPC |
| **Transit Gateway** | Connects many VPCs/networks together |
| **VPN Gateway** | VPC ↔ on-premises through VPN |
| **Direct Connect** | Dedicated private connection from on-premises → AWS |
| **ENI** | Network interface connecting a resource to a VPC |
- **Private network in AWS** → VPC
- **Separate IP ranges within a VPC** → subnets
- **Internet access from a public subnet** → Internet Gateway (+ route + public IP)
- **Outbound internet from a private subnet** → **NAT Gateway**
- **Stateful firewall, allow only, at the instance** → Security Group
- **Stateless firewall, allow and deny, at the subnet** → NACL
- **Block a specific IP address** → **NACL** (security groups cannot deny)
- **Inbound allowed but the response is blocked** → **NACL outbound rules**
- **EC2 cannot connect** → check both security group and NACL
- **See why traffic is blocked** → **VPC Flow Logs** (look for REJECT)
- **Know who called an AWS API** → **CloudTrail**
- **Track resource configuration changes** → AWS Config
- **Private access to S3 or DynamoDB** → **Gateway endpoint** (free)
- **Private access to other AWS services** → **Interface endpoint** (PrivateLink)
- **Connect two VPCs privately** → VPC peering (**no transitive routing**)
- **High availability** → subnets in multiple AZs
- **A subnet lives in exactly one AZ; a VPC spans AZs**
