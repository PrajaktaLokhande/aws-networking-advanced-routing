# AWS Networking Lab — Advanced Routing (EC2 Firewall/Router + DMZ)

Practice **advanced VPC routing** with a **DIY firewall/router** EC2 instance in a **DMZ subnet**.  
You’ll disable **source/dest check**, add a **second ENI**, and route the **private subnet’s default route** to the firewall, then test flows and capture packets.

> ⚠️ **Costs**: Uses an Elastic IP and multiple EC2 instances. Delete the stack when done.

---

## 🗺️ Architecture Diagram

```mermaid
flowchart LR
  subgraph AWS["VPC 10.70.0.0/16"]
    subgraph PUB["Public 10.70.1.0/24"]
      BAST[(Bastion EC2\nPublic IP)]
      RT_PUB[RT: 0.0.0.0/0 → IGW]
    end

    subgraph DMZ["DMZ 10.70.2.0/24"]
      FW[(Firewall EC2\neth0 in DMZ (public)\neth1 in Private)]
      RT_DMZ[RT: 0.0.0.0/0 → IGW]
    end

    subgraph PRIV["Private 10.70.3.0/24"]
      APP[(App EC2\nNo public IP)]
      RT_PRIV[RT: 0.0.0.0/0 → FW (eni-*)]
    end

    IGW[Internet Gateway]
  end

  BAST --- RT_PUB --- IGW
  FW --- RT_DMZ --- IGW
  APP --- RT_PRIV -.-> FW
```

**Key idea**: The **private subnet** sends **all egress** to the **firewall instance** (target = **NetworkInterfaceId** of FW’s **private‑side ENI**). The firewall does **IP forwarding + NAT** out via its **DMZ/public** interface.

---

## 📦 What gets deployed

- **VPC 10.70.0.0/16** with three subnets:
  - **Public** `10.70.1.0/24` (0/0 → IGW)
  - **DMZ** `10.70.2.0/24` (0/0 → IGW)
  - **Private** `10.70.3.0/24` (0/0 → **Firewall ENI**)
- **EC2 instances (Amazon Linux 2023)**
  - **Bastion** (Public) — SSH entry point
  - **Firewall** (DMZ) — **two ENIs** (DMZ + Private), **source/dest check disabled**, **IP forwarding + NAT** enabled
  - **App** (Private) — simple HTTP server
- **Security Groups**
  - Bastion SG: SSH(22) from **YourIpCidr**
  - Firewall SG: SSH(22) from **YourIpCidr**; allow forwarding between subnets (we keep it open to VPC CIDR)
  - App SG: HTTP(80) from VPC CIDR; SSH from Bastion/Firewall
- **Elastic IP** on the Firewall’s DMZ interface (for stable public egress)

---

## 🔧 Parameters

- `KeyName` — EC2 key pair for SSH
- `YourIpCidr` — your IP in CIDR (e.g., `203.0.113.10/32`) for SSH
- `InstanceType` — default `t3.micro`

---

## 🚀 Deploy

1. AWS Console → **CloudFormation** → **Create stack** → *With new resources*.  
2. Upload `cloudformation/advanced-routing-firewall.yaml`.  
3. Set parameters and **Create stack**.  
4. Wait for **CREATE_COMPLETE** → open **Outputs** for:
   - `BastionPublicIP`, `FirewallPublicIP`, `AppPrivateIP`

---

## ✅ Lab Steps

### 1) Validate routing via firewall
From your laptop (to bastion):
```bash
ssh -i /path/to/key.pem ec2-user@<BastionPublicIP>
```

From bastion → private app (through the firewall path):
```bash
curl -I http://10.70.3.10   # or the actual AppPrivateIP
```

From the **App** (private) to the Internet (egress through firewall NAT):
```bash
ssh ec2-user@10.70.3.10
curl -I https://aws.amazon.com
```

> If you get HTTP 200/301 headers, your **private egress via FW NAT works**.

### 2) Packet capture on the firewall (DMZ instance)
On the firewall host:
```bash
ssh -i /path/to/key.pem ec2-user@<FirewallPublicIP>

# Watch forwarded packets (adjust device names if needed: eth0=DMZ, eth1=Private)
sudo tcpdump -ni eth1 host 10.70.3.10
sudo tcpdump -ni eth0 dst port 80 or dst port 443
```

Trigger traffic again from the App or Bastion and **observe flows** (before/after NAT).

### 3) Break and fix the route (optional)
- Edit **Private RT** default route to **IGW** (incorrect) → private instance loses egress.  
- Restore route to **FW ENI** → traffic recovers.

### 4) Firewall rules (optional hardening)
The template permits all forwarding; try tightening with `iptables` on the FW:
```bash
# Allow established
sudo iptables -A FORWARD -m state --state ESTABLISHED,RELATED -j ACCEPT
# Allow HTTP from private to anywhere
sudo iptables -A FORWARD -s 10.70.3.0/24 -p tcp --dport 80 -j ACCEPT
# Drop the rest
sudo iptables -A FORWARD -j DROP
```

---

## 🧭 Where to look

- **VPC → Route Tables**: Private RT `0.0.0.0/0 → eni-…` (FW private ENI)  
- **EC2 → Instances**: Firewall **Source/dest check = disabled**; Firewall has **2 ENIs**  
- **EC2 → Network Interfaces**: FW ENIs in DMZ and Private  
- **EC2 → Security Groups**: Verify SGs as described

---

## 🆘 Troubleshooting

- **Private instance has no internet**  
  - Check Private RT default route points to **FW ENI**, not IGW/NAT.  
  - On FW, verify **IP forwarding** (`/proc/sys/net/ipv4/ip_forward` = 1) and MASQUERADE rule exists.  
  - Ensure FW’s DMZ interface has a public path (DMZ RT → IGW, EIP attached).
- **Can’t SSH to instances**  
  - Confirm `YourIpCidr` is correct; corporate networks may block outbound 22.
- **tcpdump shows nothing**  
  - Use `ip a` on the FW to confirm interface names (Amazon Linux 2023 often uses `eth0`, `eth1`). Capture on the correct one.

---

## 🧹 Cleanup

CloudFormation → select the stack → **Delete** (releases the EIP and ENIs).

---

## 📁 Repo Layout

```
cloudformation/
  advanced-routing-firewall.yaml
docs/
  architecture.md
.github/workflows/
  cfn-validate.yml
README.md
LICENSE
.gitignore
```
