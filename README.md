# AWS VPC Troubleshooting Lab

I built a small VPC on AWS, deliberately broke its connectivity in four different ways, and diagnosed each failure with **VPC Flow Logs** and **VPC Reachability Analyzer**, the way a cloud support engineer works a "my instance is unreachable" ticket. Each incident is documented as a postmortem with a timeline, evidence, root cause, fix, and prevention steps.

**Author:** Piyush Awasthi

**Related project:** [AWS Incident Response Lab](https://github.com/piyush1302000/aws-incident-response-lab) — CPU spike and service outage on EC2 with CloudWatch.

---

## What I built

```
                    Internet
                        |
             [ Internet Gateway: lab-igw ]
                        |
+------------------- lab-vpc  10.0.0.0/16 -------------------+
|                                                            |
|  Public subnet 10.0.1.0/24      Private subnet 10.0.2.0/24 |
|  +--------------------+         +----------------------+   |
|  | lab-web            |  --->   | lab-private          |   |
|  | Apache, public IP  |         | app on TCP 8080      |   |
|  | SG: web-sg         |         | SG: private-sg       |   |
|  +--------------------+         +----------------------+   |
|  Route table: lab-public-rt     Route table: lab-private-rt|
|  (0.0.0.0/0 -> lab-igw)         (local route only)         |
+------------------------------------------------------------+
```

- **Network:** custom VPC, one public and one private subnet, internet gateway, separate route tables, and the VPC's default network ACL
- **Security:** `web-sg` (SSH from my IP, HTTP from anywhere) and `private-sg` (app port and ICMP allowed only from `web-sg`)
- **Compute:** two Amazon Linux 2023 EC2 instances. `lab-web` runs Apache; `lab-private` runs a small app on port 8080 and has no public IP
- **Observability:** VPC Flow Logs delivered to CloudWatch Logs, queried in CloudWatch Logs Insights, plus Reachability Analyzer paths
- **Cost design:** no NAT gateway, and the lab was built to be torn down the same day

---

## Incident reports

| # | Failure injected | Symptom | How it was found | Report |
|---|---|---|---|---|
| 1 | HTTP rule removed from the web security group | Website times out; SSH still works | Flow Logs showed `REJECT` on port 80. Reachability Analyzer returned `ENI_SG_RULES_MISMATCH` naming `web-sg` | [01-security-group-missing-http-rule.md](01-security-group-missing-http-rule.md) |
| 2 | Deny rule (50) added to the network ACL, ahead of the allow-all rule | Port 80 blocked even though the security group allows it | Reachability Analyzer returned `SUBNET_ACL_RESTRICTION` naming the ACL; the ACL's rule list showed rule 50 | [02-network-acl-deny-rule.md](02-network-acl-deny-rule.md) |
| 3 | Default route to the internet gateway deleted | Public subnet has no path to the internet | Reachability Analyzer reported the route table has no applicable route to the internet gateway | [03-missing-internet-gateway-route.md](03-missing-internet-gateway-route.md) |
| 4 | Wrong port (8081 instead of 8080) on the private tier's security group | `ping` works, the app request times out | Ping versus `curl` isolated it to one port; Flow Logs showed `REJECT` on port 8080 | [04-private-tier-wrong-port.md](04-private-tier-wrong-port.md) |

---

## Troubleshooting checklist

The order I now follow for "a user can't reach their instance":

1. **Instance:** is it running and passing status checks?
2. **Public IP:** does it have one, and does the subnet auto-assign it?
3. **Security group:** does an inbound rule allow this port from this source?
4. **Network ACL:** does it allow the inbound port *and* the return traffic on ephemeral ports (it is stateless)?
5. **Route table:** is the subnet associated with a table that has `0.0.0.0/0` → internet gateway?
6. **Internet gateway:** is it attached to the VPC?
7. **Inside the instance:** is the service listening (`ss -tlnp`), and is there an OS firewall?
8. **Tools:** Reachability Analyzer to find *which component* blocks the path, and Flow Logs to see *that* traffic was accepted or rejected.

A quick signal on the error itself: a **timeout** means packets are being dropped silently by a firewall or routing; **connection refused** means the packet arrived but nothing was listening.

---

## What I learned

- **Flow Logs and Reachability Analyzer answer different questions.** Flow Logs show that traffic was rejected, but they can't tell a security group rejection from a network ACL rejection. The analyzer names the component that blocks the path.
- **Run diagnostics with the right inputs.** An early analysis created without a destination port reported "Reachable" while HTTP was blocked, because it wasn't testing port 80. Always set the protocol and port.
- **The analyzer points to the component, not always the rule.** For the network ACL, I still had to read the ACL's rules to find the deny rule that caused it. Rules are evaluated from the lowest number, and the first match wins.
- **Security groups are stateful; network ACLs are stateless.** Network ACLs need explicit allows for return traffic on ephemeral ports.
- **A subnet is only "public" because of its route table.** Deleting one route turned the public subnet into one with no internet path, even though the association, security group, and ACL were all correct.
- **Isolate with simple tests.** Comparing `ping` with an application request split "is the network up?" from "is this port allowed?" in two commands. Watch out for `curl -s`, which hides the error message.

---

## Skills demonstrated

Amazon VPC (subnets, route tables, internet gateway, network ACLs, security groups) · VPC Flow Logs · VPC Reachability Analyzer · Amazon CloudWatch Logs and Logs Insights · Amazon EC2 and Linux (`ping`, `curl`, `systemctl`, SSH) · network troubleshooting and root-cause analysis · incident timelines and postmortem writing

---

## Repository contents

```
├── README.md
├── 01-security-group-missing-http-rule.md
├── 02-network-acl-deny-rule.md
├── 03-missing-internet-gateway-route.md
├── 04-private-tier-wrong-port.md
└── screenshots/        evidence used in the reports
```
