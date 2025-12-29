# Incident 001 — EC2 SSH Connectivity Failure (Production Access Issue)

## Ticket Summary (Customer Report)
**Title:** Unable to SSH into production EC2 instance  
**Severity:** Sev-2 (Degraded ops access; service still running)  
**Reported Symptoms:**
- SSH attempts to the instance public IP time out
- No recent code deployments, but network changes were made earlier today

**Business Impact:**
- On-call engineers cannot access the instance for maintenance
- Increased recovery time risk if service degrades

---

## Environment
- **Cloud:** AWS
- **Compute:** EC2 (Linux)
- **Network:** VPC, Subnet, Route Table, Internet Gateway
- **Security Controls:** Security Group, NACL
- **Access Method:** SSH (port 22) using key-based auth

---

## Initial Triage (What I Checked First)
1. Confirmed instance state and health checks
2. Verified correct public IP / DNS target
3. Confirmed user source IP and whether VPN/corporate egress was required
4. Checked for recent infrastructure/network changes (SG/NACL/Routes)

---

## Troubleshooting Timeline (Step-by-Step)

### Step 1 — Validate instance status
**Goal:** Ensure this is not a stopped/terminated or impaired instance.

**AWS CLI**
```bash
aws ec2 describe-instances --instance-ids i-xxxxxxxxxxxxxxxxx \
  --query "Reservations[].Instances[].{State:State.Name,PublicIp:PublicIpAddress,SubnetId:SubnetId,VpcId:VpcId}" \
  --output table

**Findings:**
- Instance state was `running`
- System and instance status checks passed
- Public IP address was assigned
- Instance was confirmed in the expected VPC and subnet

**Conclusion:**
The issue was not caused by instance state or AWS health checks. Investigation proceeded to the network access path.
