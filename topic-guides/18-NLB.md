## Network load balancer
**What it is:**
A **Layer 4** load balancer that distributes traffic based on network connections rather than HTTP content. ([docs.aws.amazon.com](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/introduction.html?utm_source=chatgpt.com))
**Supports**
- **TCP**
- **UDP**
- **TLS**
- **QUIC** ([docs.aws.amazon.com](https://docs.aws.amazon.com/elasticloadbalancing/latest/APIReference/Welcome.html?utm_source=chatgpt.com))
**Key features**
- Very high performance → millions of requests/connections
- **Static IP addresses**; can assign an Elastic IP per AZ
- Preserves client IP in supported configurations
- Supports **dynamic port mapping** for ECS
- Routes traffic to healthy targets ([docs.aws.amazon.com](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/introduction.html?utm_source=chatgpt.com))
**Use NLB when:**
> Need **Layer 4**, TCP/UDP, very high performance, or **static IPs**.
**ALB vs NLB**
> **ALB → Layer 7 → HTTP/HTTPS → path/host routing**
> **NLB → Layer 4 → TCP/UDP/TLS → high performance/static IP** ([docs.aws.amazon.com](https://docs.aws.amazon.com/elasticloadbalancing/latest/APIReference/Welcome.html?utm_source=chatgpt.com))
**Exam memory:**
**NLB = TCP/UDP + static IP + very high performance.**
![](notion-file-block://3b94583f-1b37-80a7-bafb-fd844ba3cfaa/27388301-fb20-4729-ad00-3bf6902165d9?space_id=20330d1c-37a2-435d-a071-398e6a6dd89a&name=image.png)
