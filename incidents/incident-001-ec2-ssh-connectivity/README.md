# Incident 001 — EC2 SSH Connectivity Failure (Production Access Issue)

## Ticket Summary (Customer Report)

**Title:** Unable to SSH into production EC2 instance  
**Severity:** Sev-2 (Operational access degraded; service still running)

**Reported Symptoms:**
- SSH attempts to the instance public IP resulted in connection timeouts
- No recent application deployments were reported
- Network-related configuration changes were performed earlier in the day

**Business Impact:**
- On-call engineers were unable to access the instance for maintenance or troubleshooting
- Increased operational risk if application issues occurred requiring immediate access

---

## Environment

- **Cloud Provider:** AWS
- **Compute:** EC2 (Linux)
- **Region:** Not specified (single-region deployment)
- **Networking:** VPC, Public Subnet, Route Table, Internet Gateway
- **Security Controls:** Security Groups, Network ACLs
- **Access Method:** SSH (TCP port 22, key-based authentication)

---

## Initial Triage (First Response Actions)

1. Confirmed the EC2 instance state and AWS health checks
2. Verified the public IP address and DNS target
3. Confirmed the source IP used for SSH access (VPN / corporate egress)
4. Reviewed recent infrastructure changes related to networking and security

---

## Troubleshooting Timeline (Step-by-Step Investigation)

### Step 1 — Validate instance state and health

**Goal:**  
Ensure the issue is not caused by a stopped, terminated, or impaired EC2 instance.

**AWS CLI**
```bash
aws ec2 describe-instances \
  --instance-ids i-xxxxxxxxxxxxxxxxx \
  --query "Reservations[].Instances[].{State:State.Name,PublicIp:PublicIpAddress,SubnetId:SubnetId,VpcId:VpcId}" \
  --output table

```

## Findings:

- Instance state was running
- System and instance status checks passed
- A public IP address was assigned
- Instance was located in the expected VPC and subnet

## Conclusion:
The issue was not related to EC2 instance state or AWS health. Investigation continued to the network access path.

### Step 2 — Verify subnet routing and Internet Gateway attachment

**Goal:**
Ensure the subnet containing the instance had a valid route to the internet.

**AWS CLI**
```bash
aws ec2 describe-route-tables \
  --filters "Name=association.subnet-id,Values=subnet-xxxxxxxx" \
  --query "RouteTables[].Routes[]" \
  --output table

```
## Findings:
- Subnet route table contained a default route 0.0.0.0/0
- Default route pointed to an attached Internet Gateway (igw-xxxxxxx)
- No blackhole or conflicting routes were present

## Conclusion:
Internet routing was correctly configured. The connectivity issue was not caused by missing or incorrect route table entries.

### Step 3 — Review Security Group inbound rules (SSH access)

**Goal:**
Verify that the Security Group allowed SSH access from the correct source IP range.

**AWS CLI**
```bash
aws ec2 describe-security-groups \
  --group-ids sg-xxxxxxxx \
  --query "SecurityGroups[].IpPermissions" \
  --output json

```
