# Highly Available Multi-AZ VPC Lab
**Author:** Alphonse Shema | **Region:** eu-north-1 (Stockholm) | **IaC:** AWS CloudFormation + GitSync

---

## Live Demo

| Server | Live URL | Availability Zone |
|---|---|---|
| Web Server 1 | http://13.62.76.175/ | eu-north-1a (AZ1) |
| Web Server 2 | http://13.60.209.59/ | eu-north-1b (AZ2) |

> Both servers run Apache HTTP Server installed automatically via CloudFormation UserData.
> Each page displays the author name, lab name, and which Availability Zone it is serving from.

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
    │           AZ 1                  AZ 2       │
    │  ┌────────────────┐    ┌────────────────┐  │
    │  │  Public Subnet │    │  Public Subnet │  │
    │  │  10.0.1.0/24   │    │  10.0.2.0/24   │  │
    │  │                │    │                │  │
    │  │  WebServer1    │    │  WebServer2    │  │
    │  │  13.62.76.175  │    │  13.60.209.59  │  │
    │  │  NAT-GW-1      │    │  NAT-GW-2      │  │
    │  └───────┬────────┘    └────────┬───────┘  │
    │          │ (egress only)        │ (egress)  │
    │  ┌───────▼────────┐    ┌────────▼───────┐  │
    │  │ Private Subnet │    │ Private Subnet │  │
    │  │  10.0.3.0/24   │    │  10.0.4.0/24   │  │
    │  │                │    │                │  │
    │  │  AppServer1    │    │  AppServer2    │  │
    │  │  No public IP  │    │  No public IP  │  │
    │  └────────────────┘    └────────────────┘  │
    └────────────────────────────────────────────┘
```

---

## What This Template Deploys

| Resource | Count | Details |
|---|---|---|
| VPC | 1 | 10.0.0.0/16, DNS hostnames and DNS support enabled |
| Internet Gateway | 1 | Attached to VPC, enables public internet access |
| Public Subnets | 2 | One per AZ, auto-assign public IP on launch |
| Private Subnets | 2 | One per AZ, no public IP assigned |
| NAT Gateways | 2 | AZ-aligned, one per public subnet |
| Elastic IPs | 2 | One per NAT Gateway, static public IPs |
| Route Tables | 3 | 1 public shared, 2 private AZ-specific |
| Security Groups | 2 | Web tier (HTTP + ICMP), App tier (ICMP only) |
| IAM Role | 1 | AmazonSSMManagedInstanceCore for Session Manager |
| Instance Profile | 1 | Wraps IAM role for EC2 attachment |
| Web EC2 Instances | 2 | Public subnets, Apache via UserData, t3.micro |
| App EC2 Instances | 2 | Private subnets, no public IP, t3.micro |

---

## Repository Structure

```
vpc-lab/
├── vpc-lab.yaml       # Main CloudFormation template
└── README.md          # This file
```

---

## Deploy the Stack

### Option 1 - GitSync (Recommended)

GitSync automatically updates the stack every time you push a change to GitHub.

1. Push `vpc-lab.yaml` to your GitHub repo on the `main` branch
2. Go to **CloudFormation -> Create stack -> Sync from Git**
3. Connect to your GitHub repo and select `vpc-lab.yaml`
4. Stack name: `vpc-lab-moodle-stack`
5. IAM role: `vpc-lab-cloudformation-role`
6. Tick the acknowledgement checkbox at Step 4:

```
I acknowledge that AWS CloudFormation might create IAM resources with custom names.
```

7. Click **Submit**

### Option 2 - AWS CLI

```bash
aws cloudformation create-stack \
  --stack-name vpc-lab-moodle-stack \
  --template-body file://vpc-lab.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --region eu-north-1
```

> Full deployment takes approximately 5-10 minutes.
> NAT Gateways are the slowest resource and take 3-5 minutes alone.

---

## Validation Tests

After the stack shows CREATE_COMPLETE, go to **CloudFormation -> Outputs**
and copy all IP values before running these tests.

### Test 1 - Apache Web Servers

Open both URLs in your browser and confirm the page loads with your name and AZ label:

| Server | URL |
|---|---|
| Web Server 1 | http://13.62.76.175/ |
| Web Server 2 | http://13.60.209.59/ |

### Test 2 - Session Manager Access

```
EC2 -> Instances -> select any instance -> Connect -> Session Manager -> Connect
```

No SSH key required. A terminal opens directly in your browser.

### Test 3 - Public to Private Ping

From a **WebServer** Session Manager session:

```bash
ping -c 4 <AppServer1PrivateIP>
ping -c 4 <AppServer2PrivateIP>
```

Expected: ping replies from 10.0.3.x and 10.0.4.x

### Test 4 - Private Instance Outbound Internet via NAT Gateway

From an **AppServer** Session Manager session:

```bash
# Basic connectivity
ping -c 4 8.8.8.8

# Confirm NAT Gateway is the exit point (appears as first hop)
traceroute 8.8.8.8

# Full outbound test
sudo yum install -y curl
curl -s https://example.com | head -5
```

Expected: all commands succeed and traceroute shows NAT Gateway IP as first hop.

---

## Security Design

| Rule | Web Tier | App Tier |
|---|---|---|
| Inbound HTTP port 80 | Allowed from internet | Blocked |
| Inbound SSH port 22 | Blocked | Blocked |
| Inbound ICMP | Allowed from VPC only | Allowed from VPC only |
| Outbound internet | Allowed via IGW | Allowed via NAT Gateway |
| Public IP assigned | Yes | No |

---

## NAT Gateway Design

### Why Two NAT Gateways

This architecture uses one NAT Gateway per AZ - called AZ-aligned design:

```
PrivateSubnet-AZ1  ->  PrivateRouteTable1  ->  NatGateway1 (AZ1)  ->  IGW  ->  Internet
PrivateSubnet-AZ2  ->  PrivateRouteTable2  ->  NatGateway2 (AZ2)  ->  IGW  ->  Internet
```

Benefits:
- If AZ1 fails, AZ2 private instances keep their internet access through NatGateway2
- No cross-AZ data transfer charges - traffic stays in its own AZ
- True high availability with no single point of failure

### Regional NAT Gateway - The Alternative

AWS offers a Regional NAT Gateway that spans all AZs automatically.

| Feature | AZ-Scoped (this lab) | Regional NAT Gateway |
|---|---|---|
| Gateways needed | One per AZ (2 total) | One for the whole region |
| High availability | You manage it manually | AWS manages automatically |
| Cross-AZ traffic | None | Possible |
| Cost | 2x NAT hourly charges | 1x NAT hourly charge |

To replace this architecture with a Regional NAT Gateway:
- Remove `NatGateway2` and `NatGateway2EIP` from the template
- Keep only `NatGateway1` in `PublicSubnet1`
- Point both `PrivateRouteTable1` and `PrivateRouteTable2` to `NatGateway1`
- AWS handles AZ failover automatically

---

## Cost

> NAT Gateways are NOT covered by the AWS free tier.
> Delete the stack when not in use to avoid unnecessary charges.

| Resource | Rate | Approx Daily Cost |
|---|---|---|
| NAT Gateway x2 | $0.045 per hour each | ~$2.16 |
| t3.micro EC2 x4 | $0.0104 per hour each | ~$1.00 |
| Elastic IPs | Free while attached to NAT | $0.00 |
| **Total** | | **~$3.16 per day** |

> t3.micro EC2 instances are free tier eligible for the first 12 months.
> NAT Gateways are never free tier eligible regardless of account age.

### Delete the Stack When Not in Use

```
CloudFormation -> vpc-lab-moodle-stack -> Delete stack
```

After deletion confirm these are all removed:

```
EC2 -> NAT Gateways  -> both deleted
EC2 -> Elastic IPs   -> both released
EC2 -> Instances     -> all 4 terminated
```

---

## Author

**Alphonse Shema**
AWS Highly Available VPC Lab
Region: eu-north-1 (Stockholm)