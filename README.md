# Hospital Secure Reports — AWS CloudFormation + Private SNS Delivery

A healthcare-themed project demonstrating how to securely deliver patient
report notifications using **Infrastructure as Code** and **private AWS
networking** — messages never traverse the public internet.

## Problem Statement

How to secure patient records online and send them privately to the
intended party. A hospital needs to publish report-ready notifications to
patients via email/mobile push, while keeping the entire delivery path
private and auditable.

## Architecture

```
                        ┌─────────────────────────────────────────┐
                        │         CloudFormation Stack             │
                        │      (hospital-secure-reports-stack)     │
                        └─────────────────────────────────────────┘
                                          │
                                          ▼
┌───────────────────────────────────────────────────────────────────────┐
│                          VPC (10.0.0.0/16)                             │
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────┐     │
│   │              Public Subnet (10.0.1.0/24)                     │     │
│   │                                                               │     │
│   │   ┌───────────────────┐        ┌───────────────────────┐     │     │
│   │   │   EC2 Instance     │        │  SNS Interface         │     │     │
│   │   │  (Ubuntu 22.04)    │◄──────►│  VPC Endpoint          │     │     │
│   │   │  IAM Role:         │  HTTPS │  (Private DNS Enabled) │     │     │
│   │   │  sns:Publish only  │  :443  │                        │     │     │
│   │   └─────────┬──────────┘        └───────────┬────────────┘     │     │
│   │             │                                │                  │     │
│   └─────────────┼────────────────────────────────┼──────────────────┘     │
│                 │ SSH (management only)          │ Private connection    │
│                 ▼                                ▼                       │
│         ┌───────────────┐                                                │
│         │ Internet       │                                                │
│         │ Gateway        │                                                │
│         └───────────────┘                                                │
└───────────────────────────────────────────────────────────────────────┘
                                          │
                                          │ (private, within AWS network —
                                          │  never touches public internet)
                                          ▼
                              ┌───────────────────────┐
                              │   Amazon SNS Topic     │
                              │  hospital-reports-topic│
                              └───────────┬───────────┘
                                          │
                                          ▼
                              ┌───────────────────────┐
                              │  Email Subscription    │
                              │   (the "patient")      │
                              └───────────────────────┘
```

**The key privacy mechanism:** normally, an EC2 instance calling the SNS API
sends that traffic out through the Internet Gateway to reach AWS's public
SNS endpoint — encrypted, but still routed over the public internet. By
provisioning an **Interface VPC Endpoint** for SNS with Private DNS enabled,
SNS API calls are transparently routed through a private connection inside
AWS's own network instead. No application code changes are needed — DNS
resolution for the SNS endpoint automatically points to the private address.

## Tech Stack

- **AWS CloudFormation** — the entire VPC, subnets, security groups, IAM
  role, and EC2 instance are defined as code in a single YAML template —
  no manual console clicking for infrastructure provisioning
- **AWS VPC** — custom network with public subnet, Internet Gateway, and
  route table
- **AWS SNS (Simple Notification Service)** — pub/sub messaging for
  report-ready notifications
- **AWS Interface VPC Endpoint** — private connectivity to SNS
- **AWS IAM** — least-privilege role scoped to `sns:Publish` on a single
  named topic only
- **EC2** — Ubuntu 22.04, AWS CLI

## What This Project Demonstrates

- **Infrastructure as Code**: the full network stack (VPC, subnet, IGW,
  route table, two security groups, IAM role, EC2 instance) is provisioned
  from a single CloudFormation template — repeatable and version-controlled
- **Private service connectivity**: using an Interface VPC Endpoint so
  sensitive message traffic never leaves the AWS private network
- **Least-privilege IAM**: the EC2 instance's role can *only* publish to
  one specific SNS topic — nothing else
- **Verified privacy, not just configured**: traffic through the VPC
  Endpoint was confirmed via CloudWatch metrics (Active Connections, Bytes
  Processed) captured at the exact moment a message was published,
  providing measurable proof the private path was actually used
- **Healthcare-relevant security patterns**: data privacy and least
  privilege are directly applicable to compliance-conscious industries
  like healthcare (HIPAA-adjacent thinking, even though this is a
  learning project, not a compliant production system)

## How It Works

1. An SNS topic (`hospital-reports-topic`) is created and a subscriber
   (representing a patient) confirms their subscription
2. A CloudFormation stack deploys the VPC, EC2 instance, IAM role, and the
   SNS Interface VPC Endpoint in one operation
3. The EC2 instance uses the AWS CLI to publish a message to the SNS topic
4. Because Private DNS is enabled on the VPC Endpoint, this API call is
   automatically routed privately rather than over the public internet
5. SNS delivers the message to the subscribed email address
6. CloudWatch metrics on the VPC Endpoint confirm real traffic flowed
   through the private connection at the moment of publishing

## Screenshots

See the `/screenshots` folder for the full walkthrough — from SNS topic
creation through CloudFormation deployment, VPC Endpoint verification,
message publishing, delivery confirmation, and the CloudWatch monitoring
graph proving private connectivity.

## Key Files

- `hospital-vpc-stack.yaml` — the complete CloudFormation template
  (VPC, subnet, IGW, route table, security groups, IAM role, SNS
  VPC Endpoint, EC2 instance)

## Skills Demonstrated

AWS CloudFormation, VPC design, Interface VPC Endpoints, SNS, IAM
least-privilege policies, EC2, AWS CLI, network security, private
connectivity patterns

## Author

Burhan Bashir Hakim — transitioning into Cloud & DevOps Engineering
[LinkedIn](https://www.linkedin.com/in/burhan-bashir-hakim-48a8a3225)
