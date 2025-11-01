# OPNsense + OpenVPN on AWS — Architecture

## High level architecture

```mermaid
flowchart TB
  subgraph Internet
    direction TB
    User[Remote user with OpenVPN client]
  end

  subgraph AWS_VPC["VPC: 10.0.0.0/16"]
    direction TB
    subgraph Public_Subnet["Public subnet: 10.0.1.0/24"]
      OPN_PUB[OPNsense EC2 (ENI-public) <br/>EIP]
      NAT[NAT Gateway]
    end

    subgraph Private_Subnet["Private subnet: 10.0.2.0/24"]
      API1[API EC2 #1<br/>10.0.2.10]
      API2[API EC2 #2<br/>10.0.2.11]
      API3[API EC2 #3<br/>10.0.2.12]
    end

    IGW[Internet Gateway]
  end

  User -->|OpenVPN over Internet| OPN_PUB
  IGW --> NAT
  OPN_PUB -.->|routes VPN clients| API1 & API2 & API3
  API1 & API2 & API3 -->|outbound access| NAT

  graph LR
    ClientVPN[VPN client IP: 10.8.0.0/24]
    OPN_Private[OPNsense private ENI: 10.0.1.10]
    PrivateRT[Private subnet route table]
    API_Subnet[10.0.2.0/24 - API instances]
    AWS_VPC_Route[VPC route: 10.8.0.0/24 -> OPNsense instance]
  
    ClientVPN --> OPN_Private
    OPN_Private --> API_Subnet
    API_Subnet -->|reply| ClientVPN
    PrivateRT --> AWS_VPC_Route
```


---

# 2) AWS resources required (summary)

- VPC (10.0.0.0/16)
  - Public subnet (e.g. 10.0.1.0/24)
  - Private subnet (e.g. 10.0.2.0/24)
- Internet Gateway (attached to VPC)
- NAT Gateway (in public subnet) + Elastic IP (for private instances outbound internet access)
- Route Tables:
  - Public RT (0.0.0.0/0 -> IGW)
  - Private RT (0.0.0.0/0 -> NAT Gateway)
  - Add a specific route `10.8.0.0/24 -> <OPNsense instance>` so the private subnet replies to VPN clients (explained below)
- Security Groups:
  - SG-OPN-PUB: allow OpenVPN port (e.g. UDP 1194) from Internet, SSH from admin IP only, and allow inbound from the VPN network to private subnet
  - SG-API: allow traffic only from the private subnet and from the OPNsense private ENI/VPN network
- EC2 Instances:
  - OPNsense EC2 (2 ENIs: public + private). Elastic IP on public ENI. Disable Source/Dest checks on the instance.
  - 3 API EC2 instances in private subnet (no public IPs)
- S3 bucket (temporary) + VM Import/Export usage if you import OPNsense OVA (instructions provided)
- IAM role/policies to support VM import (if you import image)

---

# 3) Step-by-step instructions

I'll break this into two high-level parts:
- **A. Create AWS networking and API EC2 instances**
- **B. Deploy OPNsense on EC2 + configure OpenVPN + local user DB**

> I give both Console (GUI) and CLI hints where appropriate. If you prefer Terraform/CloudFormation, I can convert later.

---

## A — Create the AWS network and private API EC2s

1. **Create a VPC**
   - Console: VPC > Create VPC.
   - CIDR: `10.0.0.0/16`. Name it `dev-opnsense-vpc`.

2. **Create subnets**
   - Public subnet: `10.0.1.0/24` (Availability Zone A). Name: `public-subnet`.
   - Private subnet: `10.0.2.0/24`. Name: `private-subnet`.

3. **Create and attach an Internet Gateway**
   - VPC > Internet Gateways > Create > Attach to your VPC.

4. **Create a public Route Table**
   - Create route table, associate with `public-subnet`.
   - Add route `0.0.0.0/0` -> Internet Gateway (IGW).

5. **Create a NAT Gateway** (so private API instances can access the internet for updates)
   - Go to NAT Gateways > Create NAT Gateway in the public subnet.
   - Allocate a new Elastic IP (EIP) for it.
   - Update the private subnet's route table: `0.0.0.0/0` -> NAT Gateway.

6. **Security groups**
   - `sg-opnsense-public`
     - Inbound:
       - `UDP 1194` or `TCP/UDP 1194` (OpenVPN) from `0.0.0.0/0` (or restrict to known source IPs if possible).
       - `TCP 22` (SSH) from your admin IP only (optional).
     - Outbound: allow all.
   - `sg-apis-private`
     - Inbound:
       - `TCP 80/443` or whatever your API ports are **from** the VPC private CIDR (`10.0.0.0/16`) **and** from the VPN subnet (we will use `10.8.0.0/24`).
       - or: allow inbound from the OPNsense private ENI IP (`10.0.1.10`) and from `10.8.0.0/24` (VPN clients).
     - Outbound: allow all (or restricted to NAT).

7. **Launch 3 EC2 instances (API servers) in the private subnet**
   - Use Ubuntu or Amazon Linux AMI.
   - Make sure "Auto-assign Public IP" is **disabled** for these instances.
   - Attach `sg-apis-private`.
   - Give them private IPs (e.g., 10.0.2.10, .11, .12) — you can set static private IPs at launch.
   - Install your API code (or simple test server) and confirm it serves on the internal IP and port.

8. **Verify private instances can access the internet**
   - From an API EC2, `curl http://ipinfo.io/ip` (or `sudo apt update`) — traffic should go via NAT Gateway.

---

## B — Deploy OPNsense on EC2 & configure OpenVPN

> **Important note about OPNsense on AWS**: OPNsense is a FreeBSD appliance distributed as an OVA/ISO. AWS does not have an official OPNsense AMI. The standard route is to **import the OPNsense OVA** into AWS as an AMI using **VM Import/Export**, or to use a trusted community AMI. Importing an OVA requires several steps (download OPNsense image, upload to S3, use `aws ec2 import-image`). This is a one-time step I list below. If you'd rather avoid the image import complexity, you can run OpenVPN on a standard Linux EC2 and configure routing (I can provide that alternate path on request). Below I cover the OPNsense import route.

### B.1 Prepare to import OPNsense (VM Import/Export)
(If you already have an OPNsense AMI or community AMI you trust, skip to B.2)

1. **Download the OPNsense OVA**
   - From the OPNsense download site (choose a stable release OVA). Save locally (file like `OPNsense-23.7-OpenSSL-serial.ova`).
   - (I can't supply the file; download yourself.)

2. **Create an S3 bucket**
   - S3 > Create bucket `opnsense-import-<yourname>-yyyy`.

3. **Upload the OVA to the S3 bucket** (Console or `aws s3 cp`).
   - Example:
     ```bash
     aws s3 cp OPNsense-*.ova s3://opnsense-import-<yourname>-yyyy/
     ```

4. **Create an IAM role and policy for VM Import**
   - AWS has a managed policy `vmimport` example. Create an IAM role/user with permissions to `s3:GetObject`, `s3:ListBucket`, and `ec2:ImportImage`, etc. (You can follow AWS docs "VM Import/Export" for exact policy.)

5. **Run `import-image` to create an AMI**
   - Example (simplified):
     ```bash
     aws ec2 import-image \
       --description "OPNsense import" \
       --disk-containers "file://containers.json"
     ```
   - `containers.json` example:
     ```json
     [
       {
         "Description": "OPNsense OVA",
         "UserBucket": {
           "S3Bucket": "opnsense-import-<yourname>-yyyy",
           "S3Key": "OPNsense-23.7.ova"
         }
       }
     ]
     ```
   - Monitor the import task: `aws ec2 describe-import-image-tasks --import-task-ids <id>`
   - When complete you'll get an AMI ID.

> If `import-image` fails / seems too hard, consider: use a small EC2 running Linux + OpenVPN server (easy and actively supported on AMIs). I can give a step-by-step for that alternative.

### B.2 Launch OPNsense EC2 with two ENIs (public + private)

1. **Create two network interfaces**
   - ENI-PUBLIC (attach to public subnet), private IP e.g. `10.0.1.10`, primary ENI — this will be the public-facing interface.
   - ENI-PRIVATE (attach to private subnet), private IP e.g. `10.0.2.5` or better attach private ENI in **public subnet**? Wait — we want OPNsense to be in both subnets so it can route. Recommended arrangement:
     - Put OPNsense public ENI in the **public subnet** (`10.0.1.x`) — this gets the EIP.
     - Put OPNsense private ENI in the **private subnet** (`10.0.2.x`) — this gives OPNsense direct access to APIs.
   - Alternative: keep OPNsense both ENIs in the public subnet and route from private-subnet to OPNsense instance ID — but placing the private ENI in the private subnet is cleaner.

2. **Launch the OPNsense EC2 instance**
   - Use the AMI from import or community AMI.
   - Choose instance type `t3.small` or `t3.medium` (depending on VPN load).
   - When configuring network interfaces:
     - Attach ENI-PUBLIC as eth0 (public subnet) — enable `Auto-assign Public IP`? (You’ll attach an EIP to eth0 instead).
     - Add ENI-PRIVATE as eth1 (private subnet).
   - Attach `sg-opnsense-public` to the instance (this SG must be attached to the public interface).
   - **Important**: After launch, **disable Source/Destination checks** on the OPNsense instance:
     - EC2 Console > Instances > select instance > Actions > Networking > Change Source/Dest. Check => Disable.
     - This allows the instance to route traffic for other addresses.

3. **Attach an Elastic IP (EIP) to OPNsense public ENI**
   - Allocate an EIP and associate it with the public interface.

4. **Adjust route tables** (so private subnet replies to VPN clients)
   - Edit the **private subnet's route table**:
     - Add a route for the **VPN network** (example `10.8.0.0/24`) pointing to the **OPNsense instance** (select the instance ID as the target).
     - Why? If OPNsense assigns VPN clients addresses in 10.8.0.0/24 and OPNsense is doing routing (not NAT), private instances must know to send replies for 10.8.0.0/24 back to the OPNsense instance. Setting this route ensures correct return path.

   - If instead you configure OPNsense to **NAT** traffic from VPN clients to the private ENI IP (i.e., SNAT), you can avoid the VPC route addition. I recommend the route method (no NAT) — more transparent.

5. **Security Group rules recap**
   - `sg-opnsense-public` inbound:
     - `UDP 1194` (OpenVPN default) from client IP ranges or `0.0.0.0/0` (during testing).
     - `TCP 22` from admin IP (optional).
     - `ICMP` from admin IP (optional).
   - `sg-apis-private` inbound:
     - Allow API ports from `10.0.0.0/16` and `10.8.0.0/24` (VPN subnet) — or from OPNsense private ENI IP.

6. **Initial access to OPNsense**
   - Open browser to `https://<EIP>` — OPNsense GUI default login is usually `root/opnsense` or `installer`. Follow OPNsense initial setup to set the admin password and network interfaces (confirm eth0 public, eth1 private).
   - Verify the private interface can reach the API machines (e.g., from OPNsense Diagnostics > Ping to the API private IPs).

---

### B.3 Configure OpenVPN in OPNsense (local user DB, no MFA)

1. **Create a Certificate Authority (CA) and Server certificate**
   - In OPNsense GUI: `System > Trust > Authorities` > Add (create a new internal CA).
   - `System > Trust > Certificates` > Create a server certificate for the OpenVPN server (signed by your CA).

2. **Enable the OpenVPN server**
   - `VPN > OpenVPN > Servers` > Add.
   - Type: choose `Local User Access` (this uses local authentication).
   - Protocol: UDP (recommended) on port `1194` (or custom).
   - Interface: `WAN` (public interface).
   - Backend for authentication: choose `Local Database`.
   - Server Mode / Topology: choose `Remote Access (SSL/TLS + User Auth)` or `Remote Access (SSL/TLS)` with `Local user` as the auth backend — pick the recommended one in UI.
   - Tunnel network: `10.8.0.0/24` (VPN clients).
   - Local network(s) to be accessible: `10.0.2.0/24` (private API subnet) — pushes route to clients or ensure server pushes route.
   - Save and Apply.

3. **Create local user accounts**
   - `System > Access > Users` > Add.
   - Create username/password for each developer (no MFA). Under **Certificates**, create/assign a user certificate if your server setup requires client certs (for simplicity, you can use username/password with TLS server cert + user cert optional).
   - Make sure users have shell or appropriate privileges? (Not required for OpenVPN auth.)

4. **Client Export**
   - Install/enable the OpenVPN Client Export Utility in OPNsense (if present).
   - `VPN > OpenVPN > Client Export`. Select the server and export `.ovpn` client files (these contain server address, port, certs, and pushed routes).
   - Give the exported `.ovpn` file to each user. They import it into their OpenVPN client (OpenVPN GUI, Tunnelblick on mac, etc.)

5. **Routing vs NAT**
   - If you set up route `10.8.0.0/24 -> OPNsense` in the private subnet route table (as earlier), when a VPN client 10.8.x.x hits an API there is a direct route back to OPNsense, no NAT required.
   - Alternatively, enable "IPv4 Remote Networks" and configure OPNsense to NAT VPN client traffic to OPNsense private address — easier since you may not need to edit VPC routes. For beginner simplicity: use NAT on OPNsense for VPN clients (enable `NAT` for VPN -> LAN), but that hides client addresses.

6. **Test flow**
   - From outside, run OpenVPN client with exported `.ovpn`, connect.
   - From the developer machine (VPN client), ping private API IP `10.0.2.10`.
   - From the API server, check logs see that the connection arrived from typical VPN IP or NAT IP (depending on NAT choice).

---

# 4) Example low-level checks & commands

- **Disable Source/Dest check**:
  - Console: EC2 > Instances > Actions > Networking > Change Source/Dest. Check > Disable.

- **Add route for VPN subnet to OPNsense instance**:
  - VPC > Route Tables > select private-subnet route table > Edit routes > Add `Destination: 10.8.0.0/24` `Target: instance-id` (choose the OPNsense instance).
  - Note: AWS console will allow selecting an instance target for a route in a route table.

- **Ping from OPNsense GUI**:
  - Diagnostics > Ping > `10.0.2.10` (API server) — confirm reply.

---

# 5) Security & operational notes (beginner-friendly)

- **Limit the OpenVPN port**: In SG-OPN-PUBLIC restrict the OpenVPN port to developer IPs if possible. If developers have dynamic IPs, you can keep 0.0.0.0/0 but it’s less secure.
- **Admin access to OPNsense GUI**: restrict GUI access to internal network or just your admin IP.
- **Backups**: export OPNsense config regularly (`System > Configuration > Backups`).
- **High availability**: this design is single-OPNsense; for production consider a highly-available appliance setup (more complex).
- **Logs**: enable and monitor OPNsense logs for VPN connections.
- **Costs**: EC2 for OPNsense + NAT Gateway + EIP + 3 EC2s + S3 for import will incur cost. NAT Gateway hourly & data egress can be the largest surprise.
- **AMIs**: If you use a public/community AMI, ensure it is trusted and maintained. Importing the official OPNsense OVA is more controlled.

---

# 6) Troubleshooting tips

- If clients connect but can't reach APIs:
  - Check `ip route` on OPNsense and that the tunnel network is correct.
  - Confirm private subnet route table has a route for VPN subnet back to OPNsense (if you aren’t NATing).
  - On API EC2, check `sudo iptables -L -n` (if firewall running) and security groups.
  - Ensure OPNsense has firewall rules allowing traffic from the OpenVPN interface to the private LAN.
- If the OPNsense GUI is unreachable on the public IP:
  - Confirm EIP is attached to correct ENI.
  - Confirm security group allows HTTPS/TCP 443 (or GUI port) from your admin IP.
- If install/import fails:
  - Consider fallback: run OPNsense in local VM (VMware/VirtualBox) to test configs, or run OpenVPN Server on a Linux EC2 instead (I can provide that simpler guide).

---

# 7) Minimal checklist before you call this “done”

- [ ] VPC, public/private subnets, IGW, NAT gateway set up.
- [ ] Three API EC2 instances launched in private subnet (no public IPs).
- [ ] OPNsense EC2 with two ENIs (public + private), EIP attached.
- [ ] Source/Dest check disabled on OPNsense instance.
- [ ] Private route table has `10.8.0.0/24 -> OPNsense instance` (or OPNsense set to NAT).
- [ ] Security groups configured: OPNsense open only on OpenVPN port from allowed IPs, API SG allows traffic from VPN subnet.
- [ ] OPNsense configured: CA, server certificate, OpenVPN server, local users added, client `.ovpn` exported and tested.
- [ ] Developers can connect via OpenVPN client and curl/ping private API endpoints.

---

# 8) Optional improvements (future)

- Use a **bastion** host for ssh to private instances rather than exposing SSH to internet.
- Use **CloudWatch** logs & metrics for OPNsense (via syslog) for monitoring.
- Automate everything with **Terraform** to make reproducible infra.
- Use **IAM policies** + S3 bucket policies for secure OPNsense AMI import.
- Consider AWS **Client VPN** (managed) instead of self-hosted OPNsense if you want simpler AWS-native VPN — but OPNsense gives a rich firewall GUI and local user DB capability.

---

```bash
##########################################################
# Terraform: AWS VPC + OPNsense + Private EC2s + VPN Route
# Description: Builds VPC infra for OPNsense VPN + 3 private EC2s
##########################################################

provider "aws" {
  region = "us-east-1"
}

#############################
# 1. VPC and Subnets
#############################

resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true
  tags = {
    Name = "dev-opnsense-vpc"
  }
}

resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id
  tags   = { Name = "dev-igw" }
}

resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  map_public_ip_on_launch = true
  availability_zone       = "us-east-1a"
  tags = { Name = "public-subnet" }
}

resource "aws_subnet" "private" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.2.0/24"
  map_public_ip_on_launch = false
  availability_zone       = "us-east-1a"
  tags = { Name = "private-subnet" }
}

#############################
# 2. NAT Gateway
#############################

resource "aws_eip" "nat_eip" {
  domain = "vpc"
  tags   = { Name = "nat-eip" }
}

resource "aws_nat_gateway" "nat" {
  allocation_id = aws_eip.nat_eip.id
  subnet_id     = aws_subnet.public.id
  tags          = { Name = "nat-gateway" }
  depends_on    = [aws_internet_gateway.igw]
}

#############################
# 3. Route Tables
#############################

resource "aws_route_table" "public_rt" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }
  tags = { Name = "public-rt" }
}

resource "aws_route_table_association" "public_assoc" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public_rt.id
}

resource "aws_route_table" "private_rt" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.nat.id
  }
  tags = { Name = "private-rt" }
}

resource "aws_route_table_association" "private_assoc" {
  subnet_id      = aws_subnet.private.id
  route_table_id = aws_route_table.private_rt.id
}

#############################
# 4. Security Groups
#############################

resource "aws_security_group" "opnsense_sg" {
  name        = "sg-opnsense-public"
  description = "Allow OpenVPN + SSH"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "OpenVPN UDP"
    from_port   = 1194
    to_port     = 1194
    protocol    = "udp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "SSH for admin"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["YOUR_PUBLIC_IP/32"] # <-- replace this
  }

  ingress {
    description = "Allow HTTPS GUI"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["YOUR_PUBLIC_IP/32"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = { Name = "sg-opnsense-public" }
}

resource "aws_security_group" "api_sg" {
  name        = "sg-apis-private"
  description = "Allow API access from OPNsense + VPN subnet"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "API ports (80,443)"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/16", "10.8.0.0/24"]
  }

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/16", "10.8.0.0/24"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = { Name = "sg-apis-private" }
}

#############################
# 5. OPNsense Network Interfaces
#############################

resource "aws_network_interface" "opnsense_public" {
  subnet_id       = aws_subnet.public.id
  security_groups = [aws_security_group.opnsense_sg.id]
  private_ips     = ["10.0.1.10"]
  tags = { Name = "opnsense-public-eni" }
}

resource "aws_network_interface" "opnsense_private" {
  subnet_id       = aws_subnet.private.id
  security_groups = [aws_security_group.api_sg.id]
  private_ips     = ["10.0.2.5"]
  tags = { Name = "opnsense-private-eni" }
}

#############################
# 6. Elastic IP for OPNsense
#############################

resource "aws_eip" "opnsense_eip" {
  domain                    = "vpc"
  network_interface          = aws_network_interface.opnsense_public.id
  associate_with_private_ip  = "10.0.1.10"
  depends_on                 = [aws_internet_gateway.igw]
  tags = { Name = "opnsense-eip" }
}

#############################
# 7. OPNsense EC2 Instance
#############################

resource "aws_instance" "opnsense" {
  ami           = "ami-xxxxxxxxxxxx" # <-- Replace with your imported OPNsense AMI ID
  instance_type = "t3.small"
  key_name      = "your-keypair"     # <-- Replace with your key pair name

  network_interface {
    network_interface_id = aws_network_interface.opnsense_public.id
    device_index         = 0
  }

  network_interface {
    network_interface_id = aws_network_interface.opnsense_private.id
    device_index         = 1
  }

  tags = { Name = "opnsense-fw" }
}

#############################
# 8. Private API EC2 Instances
#############################

locals {
  api_private_ips = ["10.0.2.10", "10.0.2.11", "10.0.2.12"]
}

resource "aws_instance" "api" {
  count         = 3
  ami           = "ami-0fc5d935ebf8bc3bc" # Ubuntu 22.04 (example)
  instance_type = "t3.micro"
  subnet_id     = aws_subnet.private.id
  vpc_security_group_ids = [aws_security_group.api_sg.id]
  private_ip    = local.api_private_ips[count.index]
  key_name      = "your-keypair"

  tags = {
    Name = "api-server-${count.index + 1}"
  }
}

#############################
# 9. Add VPN subnet route to private route table
#############################

# This allows API servers in private subnet to route VPN traffic (10.8.0.0/24) via OPNsense instance
resource "aws_route" "vpn_to_opnsense" {
  route_table_id         = aws_route_table.private_rt.id
  destination_cidr_block = "10.8.0.0/24"
  instance_id            = aws_instance.opnsense.id

  depends_on = [aws_instance.opnsense]
}

#############################
# 10. Outputs
#############################

output "opnsense_public_ip" {
  description = "Elastic IP for OPNsense public interface"
  value       = aws_eip.opnsense_eip.public_ip
}

output "opnsense_private_ip" {
  description = "Private ENI IP for OPNsense"
  value       = aws_network_interface.opnsense_private.private_ips[0]
}

output "api_private_ips" {
  description = "Private IPs of API instances"
  value       = local.api_private_ips
}

##########################################################
# End of main.tf
##########################################################
```
