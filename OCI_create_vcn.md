# Create a VCN in Oracle Cloud Infrastructure (OCI) — Quick Guide

This document explains how to create a Virtual Cloud Network (VCN) using the OCI Console VCN Wizard and provides recommended settings for a basic VCN with internet connectivity (public + private subnets).

---

## Overview
A VCN is the virtual network in OCI where you place compute instances, load balancers, databases and other resources. The VCN Wizard simplifies creation by automatically creating:
- VCN
- Public subnet (with route to Internet Gateway)
- Private subnet (optionally with NAT)
- Internet Gateway (IGW)
- Route tables and security lists
- DHCP options

This guide covers the "VCN with Internet Connectivity" wizard flow (recommended for most setups).

---

## Step 1 — Log in to OCI Console
1. Open https://cloud.oracle.com and sign in with your OCI credentials.
2. Select the tenancy and compartment where you want to create the VCN.

---

## Step 2 — Start the VCN Wizard
1. From the main menu (☰) go to: Networking → Virtual Cloud Networks.
2. Click **Start VCN Wizard**.

---

## Step 3 — Choose a VCN Creation Option
Choose one of the three wizard options:
- VCN with Internet Connectivity (recommended)
- VCN with Public and Private Subnets
- VCN with Only Private Subnets

For this guide, choose **VCN with Internet Connectivity** and click **Start**.

---

## Step 4 — Configure VCN and Subnets
On the wizard page fill in these settings (example values provided):

- VCN Name: `MyProjectVCN`
- Compartment: choose the desired compartment
- CIDR Block for VCN: `10.0.0.0/16` (adjust if you have overlapping networks)
- Enable DNS Resolution: Yes (recommended)

Subnet configuration (example):
- Public Subnet Name: `MyPublicSubnet`
  - CIDR: `10.0.0.0/24`
  - Purpose: hosts with public IP (bastion, load balancer)
- Private Subnet Name: `MyPrivateSubnet`
  - CIDR: `10.0.1.0/24`
  - Purpose: application/back-end instances without public IPs

Leave other options at their defaults unless you need a custom route table, security lists, or different availability domain.

Click **Create** / **Finish** to provision the VCN and its components.

---

## What the wizard creates (typical)
- VCN: `MyProjectVCN` (10.0.0.0/16)
- Internet Gateway (IGW) and Route Table entry to route 0.0.0.0/0 to IGW for public subnet
- Public subnet with default security list allowing common egress and SSH (if selected)
- Private subnet and optional NAT gateway or route for outbound connectivity
- DHCP options (DNS, etc.)

---

## Post-creation checklist & recommended actions
1. Review route tables
   - Public subnet route table should contain 0.0.0.0/0 → Internet Gateway.
   - Private subnet route table should route outbound traffic to a NAT Gateway (if private instances need outbound internet).
2. Configure Security Lists / Network Security Groups (NSGs)
   - For public hosts (bastion): allow SSH (port 22) only from trusted source IPs.
   - For application hosts: limit inbound to only required ports and sources (e.g., load balancer).
3. Create/attach Internet Gateway (IGW) or NAT Gateway if the wizard didn’t create them automatically.
4. Add a NAT Gateway for private subnet outbound internet access (if needed).
5. Create Network Security Groups (NSGs) for fine-grained security instead of broad security lists.
6. Review DHCP options (DNS servers) and update if you use custom DNS.
7. Tag resources according to your organization policy.

---

## Testing the VCN
1. Launch a compute instance in the Public Subnet and assign a public IP.
2. SSH into the instance from an allowed IP to verify connectivity.
3. Launch an instance in the Private Subnet (no public IP) and verify:
   - If NAT is present, that it can reach the internet for updates (apt update / yum).
   - If no NAT, verify connectivity to other internal services.

Example SSH command (replace with your private key and public IP):
```bash
ssh -i /path/to/key.pem ubuntu@<public_ip>
```

---

## CIDR & IP planning notes
- Choose CIDR blocks that do not overlap with on-prem networks if you plan to use VPN/DRG.
- Use /16 for VCN when you plan many subnets; use /24 subnets for individual tiers/services.
- Leave spare address space for future growth.

---

## Security and best practices
- Use least-privilege security list/NSG rules.
- Use a bastion host or Session Manager rather than opening SSH to the world.
- Use compartments and tags to organize resources.
- Consider using Service Gateway for access to Oracle services without traversing the public internet.
- Enable Flow Logs and Cloud Guard / Monitoring for production environments.

---

## Troubleshooting
- If an instance cannot reach the internet, verify:
  - Subnet route table points to IGW or NAT Gateway.
  - Security lists/NSGs allow the required egress/ingress.
  - Instance has the right public/private IP configuration and boot volume is configured correctly.
- If DNS resolution fails, check DHCP options and VCN DNS settings.

---

## Example summary (quick)
1. Networking → Virtual Cloud Networks → Start VCN Wizard  
2. Choose "VCN with Internet Connectivity" → Start  
3. Name: `MyProjectVCN`, CIDR: `10.0.0.0/16` → set subnets (10.0.0.0/24 public, 10.0.1.0/24 private) → Create  
4. Review route tables, IGW, NAT, and security lists → Launch test instances and verify.

---

Replace the example names, CIDRs and compartment with your organization values. Always plan IP space and security rules before spinning up production workloads.