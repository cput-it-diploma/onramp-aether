# Kolla-Ansible Single Node Deployment Guide

This repository provides a concise summary of all the essential commands and steps required to deploy a single-node OpenStack cloud using Kolla-Ansible. The guide is based on a practical walkthrough and is suitable for anyone looking to set up a robust, production-ready OpenStack environment.

## Target Environment

- **Platform**: AWS EC2
- **Instance Type**: t2.large (2 vCPUs, 8 GB RAM)
- **Operating System**: Ubuntu 22.04 LTS
- **OpenStack Version**: Latest stable (via Kolla-Ansible)

## Prerequisites

### AWS EC2 Instance Setup

1. **Launch an EC2 Instance**:
   - Instance type: `t2.large` (minimum)
   - AMI: Ubuntu 22.04 LTS
   - Storage: At least 100 GB root volume (recommended 200 GB for production)
   - Security Group: Open necessary ports (see below)

2. **Security Group Configuration**:
   ```
   Port 22    - SSH
   Port 80    - HTTP (Horizon)
   Port 443   - HTTPS (Horizon)
   Port 5000  - Keystone API
   Port 8774  - Nova API
   Port 9292  - Glance API
   Port 9696  - Neutron API
   Port 8776  - Cinder API
   Port 6080  - noVNC
   ```

3. **Connect to Instance**:
   ```bash
   ssh -i your-key.pem ubuntu@<instance-ip>
   ```

## System Preparation

### 1. Update System Packages

```bash
sudo apt update && sudo apt upgrade -y
```

### 2. Install Required Dependencies

```bash
sudo apt install -y python3-dev libffi-dev gcc libssl-dev python3-pip python3-venv git
```

### 3. Configure Hostname

```bash
sudo hostnamectl set-hostname openstack
echo "127.0.1.1 openstack" | sudo tee -a /etc/hosts
```

### 4. Disable Firewall (or configure appropriately)

```bash
sudo ufw disable
```

### 5. Install Docker

```bash
# Install Docker dependencies
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common

# Add Docker GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Add Docker repository
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io

# Start and enable Docker
sudo systemctl enable docker
sudo systemctl start docker

# Add user to docker group
sudo usermod -aG docker $USER
newgrp docker
```

### 6. Verify Docker Installation

```bash
docker --version
docker ps
```

## Kolla-Ansible Installation

### 1. Create Virtual Environment

```bash
python3 -m venv ~/kolla-venv
source ~/kolla-venv/bin/activate
```

### 2. Install Ansible and Kolla-Ansible

```bash
pip install -U pip
pip install 'ansible>=6,<8'
pip install git+https://opendev.org/openstack/kolla-ansible@stable/2023.2
```

### 3. Create Kolla Configuration Directory

```bash
sudo mkdir -p /etc/kolla
sudo chown $USER:$USER /etc/kolla
```

### 4. Copy Configuration Files

```bash
cp -r ~/kolla-venv/share/kolla-ansible/etc_examples/kolla/* /etc/kolla
cp ~/kolla-venv/share/kolla-ansible/ansible/inventory/all-in-one .
```

### 5. Generate Passwords

```bash
kolla-genpwd
```

## Configuration

### 1. Edit Global Configuration

Edit `/etc/kolla/globals.yml`:

```bash
nano /etc/kolla/globals.yml
```

Update the following parameters:

```yaml
---
kolla_base_distro: "ubuntu"
kolla_install_type: "source"
openstack_release: "2023.2"

# Network interface configuration
network_interface: "eth0"  # Change to your primary interface
neutron_external_interface: "eth1"  # Change to your external interface (or same as network_interface)

# VIP configuration (use your instance's IP)
kolla_internal_vip_address: "YOUR_INSTANCE_IP"
kolla_external_vip_address: "YOUR_INSTANCE_IP"

# Enable services
enable_haproxy: "no"  # Disable for single node
enable_cinder: "yes"
enable_cinder_backend_lvm: "yes"
enable_nova: "yes"
enable_neutron: "yes"
enable_glance: "yes"
enable_keystone: "yes"
enable_horizon: "yes"

# Neutron configuration
neutron_plugin_agent: "openvswitch"
```

### 2. Verify Network Interfaces

```bash
ip addr show
```

Replace `eth0` and `eth1` in the configuration with your actual interface names.

### 3. Configure LVM for Cinder (Optional)

If using Cinder with LVM backend:

```bash
# Create a physical volume
sudo pvcreate /dev/xvdb  # Adjust device name

# Create volume group
sudo vgcreate cinder-volumes /dev/xvdb
```

## Deployment

### 1. Bootstrap Servers

```bash
kolla-ansible -i ./all-in-one bootstrap-servers
```

### 2. Perform Pre-deployment Checks

```bash
kolla-ansible -i ./all-in-one prechecks
```

### 3. Deploy OpenStack

```bash
kolla-ansible -i ./all-in-one deploy
```

This step may take 20-30 minutes depending on your network speed and system resources.

### 4. Post-deployment

```bash
kolla-ansible -i ./all-in-one post-deploy
```

This will generate the `admin-openrc.sh` file in `/etc/kolla/`.

## Access and Verification

### 1. Install OpenStack Client

```bash
pip install python-openstackclient python-glanceclient python-neutronclient
```

### 2. Source Admin Credentials

```bash
source /etc/kolla/admin-openrc.sh
```

### 3. Verify OpenStack Services

```bash
openstack service list
openstack endpoint list
openstack compute service list
openstack network agent list
```

### 4. Access Horizon Dashboard

Open your browser and navigate to:
```
http://<instance-ip>/
```

Login credentials:
- **Username**: admin
- **Password**: Found in `/etc/kolla/passwords.yml` under `keystone_admin_password`

```bash
grep keystone_admin_password /etc/kolla/passwords.yml
```

## Initial Configuration

### 1. Create Demo Network

```bash
# Create network
openstack network create demo-net

# Create subnet
openstack subnet create --network demo-net \
  --subnet-range 192.168.100.0/24 \
  --dns-nameserver 8.8.8.8 \
  demo-subnet

# Create router
openstack router create demo-router

# Set router gateway (if you have external network)
openstack router set --external-gateway public demo-router

# Add subnet to router
openstack router add subnet demo-router demo-subnet
```

### 2. Create Flavor

```bash
openstack flavor create --ram 512 --disk 1 --vcpus 1 m1.tiny
openstack flavor create --ram 2048 --disk 20 --vcpus 1 m1.small
openstack flavor create --ram 4096 --disk 40 --vcpus 2 m1.medium
```

### 3. Upload Sample Image

```bash
# Download Cirros image
wget http://download.cirros-cloud.net/0.6.2/cirros-0.6.2-x86_64-disk.img

# Upload to Glance
openstack image create "cirros" \
  --file cirros-0.6.2-x86_64-disk.img \
  --disk-format qcow2 \
  --container-format bare \
  --public
```

### 4. Create SSH Key Pair

```bash
openstack keypair create --public-key ~/.ssh/id_rsa.pub mykey
```

### 5. Configure Security Group

```bash
# Allow SSH
openstack security group rule create --proto tcp --dst-port 22 default

# Allow ICMP (ping)
openstack security group rule create --proto icmp default
```

## Launch Test Instance

```bash
openstack server create --flavor m1.small \
  --image cirros \
  --network demo-net \
  --key-name mykey \
  test-instance
```

Check instance status:
```bash
openstack server list
openstack console url show test-instance
```

## Useful Commands

### Container Management

```bash
# List all containers
docker ps -a

# Check container logs
docker logs <container-name>

# Restart a specific container
docker restart <container-name>
```

### Service Management

```bash
# Check service status
kolla-ansible -i ./all-in-one check

# Reconfigure services
kolla-ansible -i ./all-in-one reconfigure

# Upgrade OpenStack
kolla-ansible -i ./all-in-one upgrade
```

### Troubleshooting

```bash
# View Kolla logs
sudo ls /var/log/kolla/

# Check specific service logs
sudo tail -f /var/log/kolla/nova/nova-api.log
sudo tail -f /var/log/kolla/neutron/neutron-server.log

# Restart all services
kolla-ansible -i ./all-in-one restart
```

## Cleanup

To destroy the OpenStack deployment:

```bash
kolla-ansible -i ./all-in-one destroy --yes-i-really-really-mean-it
```

## Common Issues and Solutions

### 1. Docker Permission Denied

```bash
sudo usermod -aG docker $USER
newgrp docker
```

### 2. Network Interface Issues

Ensure the network interface names in `/etc/kolla/globals.yml` match your system:
```bash
ip addr show
```

### 3. Insufficient Resources

For t2.large instances, consider:
- Reducing enabled services
- Using binary installation type instead of source
- Increasing instance size to t2.xlarge if possible

### 4. Pre-check Failures

Review the error messages carefully. Common issues:
- Docker not running
- Incorrect network interface names
- Insufficient disk space

### 5. Port Conflicts

Check for processes using required ports:
```bash
sudo netstat -tlnp | grep <port>
```

## Performance Optimization

### 1. Adjust Docker Storage

```bash
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json > /dev/null <<EOF
{
  "storage-driver": "overlay2"
}
EOF
sudo systemctl restart docker
```

### 2. Disable Unnecessary Services

In `/etc/kolla/globals.yml`, disable services you don't need:
```yaml
enable_heat: "no"
enable_swift: "no"
enable_ceilometer: "no"
```

## Additional Resources

- [Kolla-Ansible Documentation](https://docs.openstack.org/kolla-ansible/latest/)
- [OpenStack Documentation](https://docs.openstack.org/)
- [Kolla-Ansible Quick Start](https://docs.openstack.org/kolla-ansible/latest/user/quickstart.html)

## License

MIT License - See [LICENSE](LICENSE) file for details.

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.