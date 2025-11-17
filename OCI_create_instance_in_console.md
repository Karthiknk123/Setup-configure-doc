# Steps to Create a New Instance in Oracle OCI Console

This guide walks through creating a new compute instance in Oracle Cloud Infrastructure (OCI) using the Console and verifying SSH access.

> Replace placeholders (compartment names, image names, shape choices, VCN/subnet, and SSH key paths) with values for your tenancy.

---

## 1. Log in to Oracle OCI
- Open the OCI Console: https://www.oracle.com/in/cloud/
- Sign in with your OCI credentials to access the Oracle Cloud Infrastructure dashboard.

---

## 2. Navigate to the Instances page
- From the left-hand navigation menu: Compute → Instances.
- This page lists existing instances and provides the "Create Instance" option.

---

## 3. Create a New Instance
1. Click **Create Instance**.
2. Provide a friendly, descriptive name for the instance (useful when managing multiple instances).

---

## 4. Compartment
- In the Compartment section leave it set to `Root` (or select the specific compartment you use for organizing resources).

---

## 5. Select Image and Shape
- Image: select the OS image (for example, Canonical Ubuntu 22.04).
- Shape: choose AMD or Intel and the shape best suited for your workload (this determines underlying vCPU and memory resources).

---

## 6. Adjust CPU and RAM
- Use the shape settings or sliders to configure vCPUs and memory to match your application requirements.

---

## 7. Enable Burstable Option (optional)
- If you expect variable workloads, enable the Burstable instance option so the instance can temporarily use additional resources during spikes.

---

## 8. Network Settings
- Virtual Cloud Network (VCN): pick the VCN where the instance will run.
- Subnet: choose the subnet for the instance placement.
- Private IP vs Public IP: if you need internet access/SSH from outside OCI, assign a public IP; otherwise use only a private IP for internal access.

---

## 9. SSH Key Setup
- In the SSH Keys section:
  - Upload your existing public SSH key OR let the console generate a key pair.
  - If the console generates keys, download and securely store the private key; if you upload, ensure you have the corresponding private key locally.
- Keep the private key safe — you'll use it to SSH into the instance.

---

## 10. Configure Boot Volume
- Boot Volume size: set disk size according to OS and application needs.
- Ensure enough disk space for system, apps, and logs.

---

## 11. Create the Instance
- Review settings and click **Create**.
- OCI will provision the instance — status will transition from Provisioning to Running once complete.

---

## 12. Connect (SSH) to the Instance
- After the instance shows as Running, use SSH to connect (example for Ubuntu image):
```bash
ssh -i /path/to/private_key ubuntu@<instance_public_IP>
```
- If you used a different username for your chosen image (e.g., `opc`, `ubuntu`, or another), replace `ubuntu` with the appropriate user.

- Ensure security rules (Network Security Group / Security Lists and OS firewall) allow SSH (port 22) from your client IP.

---

## 13. Post-creation checks and tips
- Verify instance health and console logs from the OCI Console.
- Confirm networking (private/public IP) and routing if you cannot connect.
- For production, apply patching, hardening, and monitoring agents as required.
- Consider using SSH agent forwarding or a bastion host for secure access patterns when creating instances without public IPs.

---

## Quick checklist
- [ ] Name and compartment set
- [ ] Image and shape selected (Ubuntu 22.04 example)
- [ ] vCPU and memory configured
- [ ] Burstable option enabled (if required)
- [ ] VCN and Subnet chosen
- [ ] Public IP assigned (if required)
- [ ] SSH key uploaded or generated and private key saved
- [ ] Boot volume size configured
- [ ] Instance provisioned and status is Running
- [ ] SSH connectivity verified
