# Highly Available Multi-AZ VPC Lab
### AWS CloudFormation | Alphonse Shema

---

## Architecture Overview

```
                   Internet
                      │
                     IGW
                      │
    ┌─────────────────┴──────────────────────────┐
    │              VPC (10.0.0.0/16)             │
    │                                            │
    │      AZ 1                  AZ 2            │
    │  ┌───────────┐        ┌───────────┐        │
    │  │  Public   │        │  Public   │        │
    │  │ 10.0.1/24 │        │ 10.0.2/24 │        │
    │  │WebServer1 │        │WebServer2 │        │
    │  │ NAT-GW-1  │        │ NAT-GW-2  │        │
    │  └─────┬─────┘        └─────┬─────┘        │
    │        │ (egress)           │ (egress)      │
    │  ┌─────▼─────┐        ┌─────▼─────┐        │
    │  │  Private  │        │  Private  │        │
    │  │ 10.0.3/24 │        │ 10.0.4/24 │        │
    │  │AppServer1 │        │AppServer2 │        │
    │  └───────────┘        └───────────┘        │
    └────────────────────────────────────────────┘
```

---

## What This Template Deploys

| Resource | Count | Details |
|---|---|---|
| VPC | 1 | 10.0.0.0/16, DNS enabled |
| Internet Gateway | 1 | Attached to VPC |
| Public Subnets | 2 | One per AZ, auto-assign public IP |
| Private Subnets | 2 | One per AZ, no public IP |
| NAT Gateways | 2 | AZ-aligned, one per public subnet |
| Elastic IPs | 2 | One per NAT Gateway |
| Route Tables | 3 | 1 public shared, 2 private (one per AZ) |
| Security Groups | 2 | Web tier (HTTP+ICMP), App tier (ICMP only) |
| IAM Role | 1 | AmazonSSMManagedInstanceCore for SSM access |
| Web EC2 Instances | 2 | Public subnets, Apache installed via UserData |
| App EC2 Instances | 2 | Private subnets, no public IP |

---

## Deploy the Stack

### Option 1: AWS Console (GitSync)
1. Push `vpc-lab.yaml` to your GitHub repo
2. CloudFormation → Create stack → Sync from Git
3. Connect to your repo and select `vpc-lab.yaml`
4. Stack name: `vpc-lab-stack`
5. Submit

### Option 2: AWS CLI
```bash
aws cloudformation create-stack \
  --stack-name vpc-lab-stack \
  --template-body file://vpc-lab.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --region eu-north-1
```

---

## Validation Tests

### Test 1 — Apache Web Servers
After stack creation, go to CloudFormation → Outputs and copy:
- WebServer1URL → open in browser → shows webpage with your name
- WebServer2URL → open in browser → shows webpage with your name

### Test 2 — SSM Session Manager Access
1. Go to EC2 → Instances
2. Select any instance → Connect → Session Manager → Connect
3. No SSH key needed

### Test 3 — Public to Private Ping
From a web server session:
```bash
ping <AppServer1PrivateIP>
ping <AppServer2PrivateIP>
```
Expected: ping replies (ICMP allowed within VPC CIDR)

### Test 4 — Private Instance Outbound Internet via NAT
From an app server session:
```bash
ping -c 4 8.8.8.8
traceroute 8.8.8.8
sudo yum install -y curl
curl https://example.com
```
Expected: all succeed (traffic routes through NAT Gateway)

---

## Security Design

| Rule | Web Tier | App Tier |
|---|---|---|
| Inbound HTTP (80) | Yes - from internet | No - blocked |
| Inbound SSH (22) | No - blocked | No - blocked |
| Inbound ICMP | Yes - from VPC only | Yes - from VPC only |
| Outbound internet | Yes - via IGW | Yes - via NAT Gateway |
| Public IP | Yes | No |

---

## NAT Gateway Design Decision

### Why Two NAT Gateways?

This architecture uses one NAT Gateway per AZ (AZ-aligned design):

- PrivateSubnet1 (AZ1) routes outbound traffic to NatGateway1 (AZ1)
- PrivateSubnet2 (AZ2) routes outbound traffic to NatGateway2 (AZ2)

Benefits:
- If AZ1 fails, PrivateSubnet2 still has internet access via NatGateway2
- No cross-AZ data transfer charges
- True high availability with no single point of failure

### What is the Regional NAT Gateway?

AWS offers a Regional NAT Gateway — a single NAT Gateway that automatically spans all AZs in a region.

| Feature | AZ-Scoped (this lab) | Regional NAT Gateway |
|---|---|---|
| Count needed | One per AZ (2 here) | One total |
| High Availability | Manual (you create 2) | Automatic |
| Cross-AZ traffic | None | Possible |
| Cost | 2x NAT hourly charges | 1x NAT hourly charge |

How it could replace this architecture:
- Delete NatGateway1 and NatGateway2
- Create one NAT Gateway with ConnectivityType: public
- Point BOTH private route tables to this single NAT Gateway
- AWS handles AZ failover automatically

---

## Repository Structure

```
vpc-lab/
├── vpc-lab.yaml       # Main CloudFormation template
└── README.md          # This file
```

---

## Author
Alphonse Shema
AWS Highly Available VPC Lab — eu-north-1 (Stockholm)
