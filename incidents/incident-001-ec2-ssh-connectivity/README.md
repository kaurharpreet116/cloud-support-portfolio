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
## Findings:
- SSH (TCP/22) was allowed, but the source CIDR did not match the current admin/corporate egress IP
- The configured CIDR corresponded to an outdated IP range

## Conclusion:
The Security Group inbound rule was misconfigured, preventing SSH access from the valid source IP.

### Step 4 — Validate Network ACL rules (defense-in-depth check)

**Goal:**
Ensure Network ACLs were not blocking inbound or outbound SSH traffic.

**AWS CLI**
```bash
aws ec2 describe-network-acls \
  --filters "Name=association.subnet-id,Values=subnet-xxxxxxxx" \
  --query "NetworkAcls[].Entries[]" \
  --output table
```

## Findings:
- No explicit deny rules for TCP/22
- Ephemeral outbound ports were permitted

## Conclusion:
Network ACLs were not contributing to the SSH connectivity issue.

### Root Cause Analysis (RCA)

## Root Cause:
The EC2 instance Security Group allowed SSH access only from an outdated source CIDR. A recent network change modified the admin egress IP range, but the Security Group was not updated accordingly.

## Resolution

### Action Taken:
- Updated the Security Group inbound rule to allow SSH (TCP/22) from the current approved admin CIDR
- Removed obsolete IP ranges to maintain least-privilege access

## AWS CLI (example)
```bash
aws ec2 authorize-security-group-ingress \
  --group-id sg-xxxxxxxx \
  --ip-permissions '[
    {
      "IpProtocol": "tcp",
      "FromPort": 22,
      "ToPort": 22,
      "IpRanges": [
        {
          "CidrIp": "X.X.X.X/32",
          "Description": "Approved admin SSH access"
        }
      ]
    }
  ]'

```

## Validation:
```bash
ssh -i <key>.pem ec2-user@<public-ip>

```
SSH access was successfully restored.

## Preventive Actions

- Replace individual IP-based rules with approved corporate/VPN CIDR blocks
- Implement change review checks for Security Group modifications
- Enable configuration monitoring to detect Security Group drift
- Document a time-bound “break-glass” SSH access procedure

# Evidence

Supporting CLI outputs, configuration screenshots, and before/after comparisons are stored in the evidence/ directory.

## Lessons Learned
- Always validate the source CIDR when troubleshooting connectivity issues
- Security Group rules are a high-signal checkpoint for SSH timeouts
- Network changes should include a validation step for operational access paths
