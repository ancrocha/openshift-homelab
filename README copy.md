# Openshift cluster 
This document will describe the openshift installation on my lab. This is for leaning purposes and simulates offline environment.

All machines for this cluster are running on my proxmox server. Some could be combined but as for learning I preferred to keep isolated.

## Architecture

![Openshift Architecture](./images/openshift_arch.jpg)

### Host Functions

- Pfsense -> internal DNS, DHCP, Load Balance, Firewall, NTP Server.
- Registry Server -> will host internal openshift images for deployment.
- Bastion Server -> openshift commands, http server (ignition)
- Proxmox Server is HP DL380 G9 that will host the VMs.


### Network Plan

Domain: ocp.lab

| Network | CIDR | Function |
|---------|------|----------|
| Home | 10.0.0.0/24 | Home network |
| OCP Machine | 192.168.100.0/24 | all ocp VMs |
| OCP Pods | 10.128.0.0/14 | Pod to Pod SDN |
| OCP services | 172.30.0.0/16 | Kubernetes service network |

#### IP assignments

| Host | Home Net | OCP Net | Role |
|------|----------|---------|------|
| Home Router | 10.0.0.1 | - | Home Gateway + DNS |
| pfSense VM | 10.0.0.50 | 192.168.100.1 | OCP router/DNS/LB |
| Bastion VM | - | 192.168.100.2 | Jump host / admin |
| Registry VM | - | 192.168.100.3 | Mirror registry |
| Bootstrap | - | 192.168.100.10 | Temp bootstrap |
| master-1 | - | 192.168.100.11 | Control Plane |
| master-2 | - | 192.168.100.12 | Control Plane |
| master-3 | - | 192.168.100.13 | Control Plane |
| worker-1 | - | 192.168.100.21 | Compute |
| worker-2 | - | 192.168.100.22 | Compute |
| worker-3 | - | 192.168.100.23 | Compute |

#### Network flow
This is a simplified network flow I generated with AI help. 
Firewalla in the picture is my home router.

![Network Flow](./images/openshift_network_flow.jpg)

### VMs dimensioning

| Name | CPU | RAM | Disk | 
|------|-----|-----|------|
| pfSense | 2 | 2 | 20 |
| bastion | 4 | 8 | 50 |
| registry | 4 | 8 | 250 |
| bootstrap | 4 | 16 | 120 |
| master 1-3 | 4 | 16 | 120 |
| worker 1-3 | 8 | 32 | 120 |


## Proxmox architecture build

### Required downloads and images
1 - RHEL 9 from red hat developers
2 - pfSense

### OCP Network
On proxmox, create the bridge

```
name: vmbr3
comment: OCP network
```

### pfSense 

1 - Create the VM on proxmox as specified by VMs dimension session.

2 - Add network adapter on ocp network. This machine needs 2 interfaces.

3 - Install pfSense. Choose WAN interface as the home network. Choose the LAN interface the OCP internal.

4 - Disable the firewall, login the GUI using WAN IP.

#### DNS settings
```
Services 
  -> DNS Resolver 
    -> General Settings 
    -> DNS Query Forwarding ✅ 
```

```
# Services → DNS Resolver (Unbound):
# Enable ✅ | Network Interfaces: LAN | DNSSEC: optional | Enable Forwarding Mode
#
# Host Overrides (add each):
# api.ocp.lab         → 192.168.100.1
# api-int.ocp.lab     → 192.168.100.1
# *.apps.ocp.lab      → 192.168.100.1   (wildcard!)
# registry.ocp.lab    → 192.168.100.3
# bastion.ocp.lab     → 192.168.100.2
# bootstrap.ocp.lab   → 192.168.100.10
# master-1.ocp.lab    → 192.168.100.11
# master-2.ocp.lab    → 192.168.100.12
# master-3.ocp.lab    → 192.168.100.13
# worker-1.ocp.lab    → 192.168.100.21
# worker-2.ocp.lab    → 192.168.100.22
# worker-3.ocp.lab    → 192.168.100.23
```

#### DHCP settings
```
# Services → DHCP Server → LAN:
# Enable ✅ | Range: .100 to .200
# Static Mappings: add MAC → IP for each OCP VM
# (get MACs from Proxmox VM hardware tab)
```

#### Firewall Rules
```
# Disable all LAN to WAN. We want an offline environment. Wait until you install registry.
# Allow Bastion to access the internet.
# Allow offline registry to access the internet.
```

#### HAProxy on pfSense
```
pfSense → System → Package Manager → Available Packages
→ Search: haproxy
→ Install → Confirm
→ Wait for install to complete


pfSense → Services → HAProxy
→ Settings tab:
    Enable HAProxy: ✅
    Maximum connections: 1000
    internal stat port: 2200
    Save
```

```
Services → HAProxy → Backend → Add
─── Backend 1: masters_api ───────────────────────
Name:        openshift_api
Mode:        basic
Server list:
  Name: master-1  Address: 192.168.100.11  Port: 6443  
  Name: master-2  Address: 192.168.100.12  Port: 6443
  Name: master-3  Address: 192.168.100.13  Port: 6443
  Name: bootstrap  Address: 192.168.100.10  Port: 6443
Check Frequency: 2000
→ Save
─── Backend 2: masters_mcs ───────────────────────
Name:        openshift_mcs
Mode:        basic
Server list:
  Name: master-1  Address: 192.168.100.11  Port: 22623   
  Name: master-2  Address: 192.168.100.12  Port: 22623 
  Name: master-3  Address: 192.168.100.13  Port: 22623 
  Name: bootstrap  Address: 192.168.100.10  Port: 22623 
Check Frequency: 2000
Health check: basic
→ Save


─── Frontend 1: api_frontend ───────────────────────
Name:        api_frontend
Listen Address: 192.168.100.1
Port: 6443
Default backend: openshift_api
→ Save
─── Frontend 2: openshift_mcs ───────────────────────
Name:        openshift_mcs
Listen Address: 192.168.100.1
Port: 22623
Default backend: openshift_mcs
→ Save


```



### Bastion Host

1 - Create the VM on proxmox as specified by VMs dimension session. CHoose the ocp network

2 - Copy the mac address from VM hardware and add to pfsense DHCP Server.

3 - Finalize the installation. make sure your IP is correct from dhcp.

4 - I install tailscale on this machine since it is behind pfsense firewall.


```
# Dowload binaries
curl -LO https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.20.10/openshift-client-linux-4.20.10.tar.gz

curl -LO https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.20.10/oc-mirror.tar.gz

curl -LO https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.20.10/openshift-install-linux.tar.gz

tar -xvf oc-mirror.tar.gz 
tar -xvf openshift-client-linux-4.20.10.tar.gz
tar -xvf openshift-install-linux.tar.gz

mkdir bin
mv kubectl bin/ && mv oc bin/ && mv openshift-install bin/ && mv oc-mirror bin/

# Check the command
andre@bastion:~$ oc version
Client Version: 4.20.10
Kustomize Version: v5.6.0

```

### Offline Registry

1 - Create the VM on proxmox as specified by VMs dimension session. CHoose the ocp network

2 - Copy the mac address from VM hardware and add to pfsense DHCP.

3 - Finalize the installation. make sure your IP is correct from dhcp and configure hostname.


#### Configuration of offline registry host
```
sudo dnf install -y wget curl jq podman openssl

# Open registry ports
sudo firewall-cmd --permanent --add-port=8443/tcp
sudo firewall-cmd --reload

# Download quay from red hat site

tar -xzvf mirror-registry-amd64.tar.gz

sudo podman login registry.redhat.io

sudo ./mirror-registry install \
  --quayHostname registry.ocp.lab \
  --quayRoot /opt/quay \
  --initPassword changeme \
  --initUser admin

sudo cp /opt/quay/quay-rootCA/rootCA.pem /etc/pki/ca-trust/source/anchors/
sudo update-ca-trust extract

# for transfer later
sudo cp /opt/quay/quay-rootCA/rootCA.pem /home/andre
sudo chown andre:andre rootCA.pem

#Using socks I can connect to quay registry using GUI

```

* I disabled the internet to all hosts at this time...  Only bastion is allowed.

### Bastion Host
```
# Copy the generated key from registry
scp andre@192.168.100.3:/home/andre/rootCA.pem   ~/registry-ca.pem
sudo cp registry-ca.pem /etc/pki/ca-trust/source/anchors/registry-ocp-lab.crt
sudo update-ca-trust
```

```
mkdir -p ~/ocp

# Copy your Red Hat pull secret to bastion
# (download from https://console.redhat.com/openshift/install/pull-secret)
# scp it from your workstation:
# scp pull-secret.json andre@192.168.100.2:~/ocp/

cat ./pull-secret | jq . > pull-secret.json

# Encode local registry credentials
echo -n 'admin:changeme' | base64 -w0
# Copy the output

# Create local registry auth
cat > ~/ocp/local-auth.json << 'EOF'
{
  "auths": {
    "registry.ocp.lab:8443": {
      "auth": "PASTE_BASE64_HERE"
    }
  }
}
EOF

# Merge with Red Hat pull secret
jq -s '.[0] * .[1]' \
  ~/ocp/pull-secret.json \
  ~/ocp/local-auth.json \
  > ~/ocp/pull-secret-merged.json

# Verify
cat ~/ocp/pull-secret-merged.json | jq .auths | jq 'keys'
# Should list both registry.redhat.io AND registry.ocp.lab:8443

```


```
cat > ~/ocp/imageset-config.yaml << 'EOF'
kind: ImageSetConfiguration
apiVersion: mirror.openshift.io/v2alpha1
mirror:
  platform:
    channels:
    - name: stable-4.20
      type: ocp
      minVersion: 4.20.0
      maxVersion: 4.20.0
EOF

```

```
# Create workspace directory
sudo mkdir -p /opt/mirror-workspace
sudo chown andre:andre /opt/mirror-workspace

mkdir -p ~/.docker
cp ~/ocp/pull-secret-merged.json ~/.docker/config.json

# Run the mirror
oc mirror --v2 \
  --config ~/ocp/imageset-config.yaml \
  --dest-tls-verify=true \
  docker://registry.ocp.lab:8443 \
  --workspace file:///opt/mirror-workspace \
  -p ~/ocp/pull-secret-merged.json

```

```
# Download images and serve as httpd server
sudo dnf install httpd -y
sudo systemctl enable httpd
sudo systemctl start httpd
sudo systemctl status httpd
grep DocumentRoot /etc/httpd/conf/httpd.conf

sudo vi /etc/httpd/conf/httpd.conf
# change the port to 8080

firewall-cmd --add-port=8080/tcp --zone=internal --permanent
firewall-cmd --reload

sudo mkdir -p /var/www/html/openshift4/images
sudo mkdir -p /var/www/html/openshift4/4.20.10/ignitions
sudo mkdir -p /var/www/html/openshift4/4.20.10/ignitions
sudo restorecon -Rv /var/www/html/openshift4
sudo tree /var/www/html/

sudo -i
cd /var/www/html/openshift4/images/
wget https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.20/latest/rhcos-4.20.11-x86_64-live-rootfs.x86_64.img
wget https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.20/latest/rhcos-4.20.11-x86_64-live-kernel.x86_64
wget https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.20/latest/rhcos-4.20.11-x86_64-live-initramfs.x86_64.img
```

```
# Configuring tftp
sudo dnf install tftp-server tftp -y
sudo systemctl start tftp
sudo systemctl enable tftp

sudo mkdir -p /var/lib/tftpboot/pxelinux.cfg

### add example of the file

```

```
# example of install config

openshift-install create ignition-configs   --dir=.   --log-level=info
```

