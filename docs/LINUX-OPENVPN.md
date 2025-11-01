# 🧭 AWS VPN Setup (Linux OpenVPN Alternative)

This guide replaces OPNsense with a simpler **Ubuntu + OpenVPN** setup using Terraform and manual configuration — ideal for beginners.

---

## 🏗️ 1. Infrastructure Overview

### **Goal**

Allow external developers to connect securely via VPN to a private AWS subnet hosting API EC2 instances, **without exposing APIs to the public internet**.

### **Architecture Diagram**

```mermaid
graph TD
  A[Developer Laptop] -- OpenVPN --> B[OpenVPN Server (Public Subnet)]
  B -->|Private Routing| C[Private Subnet]
  C --> D[API EC2 1]
  C --> E[API EC2 2]
  C --> F[API EC2 3]
```

---

## ⚙️ 2. Terraform Infrastructure

### **Terraform resources**

* 1 VPC (10.0.0.0/16)
* 1 public subnet (10.0.1.0/24)
* 1 private subnet (10.0.2.0/24)
* Internet Gateway (IGW)
* NAT Gateway (for private subnet outbound)
* Route Tables + Associations
* Security Groups (VPN + Private)
* 1 Ubuntu EC2 for OpenVPN Server (public)
* 3 EC2 API instances (private)

> ✅ Use the Terraform script from earlier — no AMI import required.

---

## 💻 3. OpenVPN Installation on Ubuntu EC2

SSH into your public EC2 instance:

```bash
ssh -i your-key.pem ubuntu@<public-ip>
```

Then run:

```bash
sudo apt update && sudo apt install openvpn easy-rsa -y
```

---

## 🔐 4. OpenVPN Server Setup (Easy-RSA)

### Initialize PKI:

```bash
make-cadir ~/openvpn-ca
cd ~/openvpn-ca
source vars
./clean-all
./build-ca
```

### Create Server Cert + Key:

```bash
./build-key-server server
./build-dh
openvpn --genkey --secret keys/ta.key
```

### Create Client Cert:

```bash
./build-key client1
```

---

## 🧱 5. Configure the OpenVPN Server

Copy sample config:

```bash
sudo gunzip -c /usr/share/doc/openvpn/examples/sample-config-files/server.conf.gz | sudo tee /etc/openvpn/server.conf
```

Edit `/etc/openvpn/server.conf`:

```bash
sudo nano /etc/openvpn/server.conf
```

Modify/add the following lines:

```ini
port 1194
proto udp
dev tun
ca ca.crt
cert server.crt
key server.key
dh dh2048.pem
tls-auth ta.key 0
server 10.8.0.0 255.255.255.0
push "route 10.0.2.0 255.255.255.0"
keepalive 10 120
cipher AES-256-CBC
user nobody
group nogroup
persist-key
persist-tun
status openvpn-status.log
verb 3
```

---

## 🌐 6. Enable IP Forwarding and NAT

### Edit sysctl:

```bash
sudo nano /etc/sysctl.conf
```

Uncomment or add:

```
net.ipv4.ip_forward=1
```

Apply immediately:

```bash
sudo sysctl -p
```

### Configure iptables NAT:

```bash
sudo iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE
sudo sh -c 'iptables-save > /etc/iptables.rules'
```

Persist across reboots (create a systemd script):

```bash
sudo tee /etc/network/if-up.d/iptables <<EOF
#!/bin/sh
iptables-restore < /etc/iptables.rules
EOF
sudo chmod +x /etc/network/if-up.d/iptables
```

---

## 🧩 7. Start and Enable OpenVPN

```bash
sudo systemctl start openvpn@server
sudo systemctl enable openvpn@server
```

Check status:

```bash
sudo systemctl status openvpn@server
```

---

## 📦 8. Create Client Config (`client1.ovpn`)

On server:

```bash
cd ~/openvpn-ca/keys
sudo cat ca.crt client1.crt client1.key ta.key > /tmp/client1-bundle.txt
```

Create base config:

```bash
sudo tee /tmp/client1.ovpn <<EOF
client
dev tun
proto udp
remote <your-public-ip> 1194
resolv-retry infinite
nobind
persist-key
persist-tun
remote-cert-tls server
cipher AES-256-CBC
verb 3
EOF
```

Append certs:

```bash
sudo bash -c 'echo "<ca>" >> /tmp/client1.ovpn && cat ~/openvpn-ca/keys/ca.crt >> /tmp/client1.ovpn && echo "</ca>" >> /tmp/client1.ovpn'
```

Then add `<cert>`, `<key>`, and `<tls-auth>` sections similarly.

Finally, download it:

```bash
scp -i your-key.pem ubuntu@<public-ip>:/tmp/client1.ovpn ./
```

---

## 🧠 9. Client Setup

Import `client1.ovpn` into your OpenVPN GUI client (Windows/Mac/Linux) or use:

```bash
sudo openvpn --config client1.ovpn
```

Once connected, test access:

```bash
ping 10.0.2.10   # Example API server in private subnet
```

---

## ✅ 10. Security Group Rules

| Resource    | Port     | Source        |
| ----------- | -------- | ------------- |
| OpenVPN EC2 | 1194/UDP | Developer IPs |
| OpenVPN EC2 | 22/TCP   | Admin IPs     |
| API EC2     | 80/443   | 10.8.0.0/24   |

---

## 🧾 Summary

✅ Simple OpenVPN on Ubuntu (no AMI import)
✅ Secure private subnet access via VPN
✅ Minimal setup — all manual and transparent
