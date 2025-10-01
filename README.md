# Kolla-Ansible Single Node Deployment Guide

This repository provides a comprehensive guide for deploying a single-node OpenStack cloud using Kolla-Ansible. This guide is based on practical implementation and is suitable for setting up a robust, production-ready OpenStack environment.

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [System Requirements](#system-requirements)
- [Environment Preparation](#environment-preparation)
- [Kolla-Ansible Installation](#kolla-ansible-installation)
- [Configuration](#configuration)
- [Deployment](#deployment)
- [Post-Deployment Verification](#post-deployment-verification)
- [Troubleshooting](#troubleshooting)
- [Useful Commands](#useful-commands)

## Overview

Kolla-Ansible is an OpenStack deployment tool that uses Ansible playbooks to deploy OpenStack services in Docker containers. This guide focuses on deploying all OpenStack services on a single node, which is ideal for development, testing, or small-scale production environments.

## Prerequisites

Before starting the deployment, ensure you have:

- A fresh installation of Ubuntu 22.04 LTS (recommended) or similar Linux distribution
- Root or sudo access to the system
- Basic understanding of Linux command line
- Familiarity with OpenStack concepts (optional but helpful)
- Stable internet connection for downloading packages and container images

## System Requirements

### Minimum Hardware Requirements

- **CPU**: 4 cores (8 cores recommended)
- **RAM**: 8 GB (16 GB or more recommended)
- **Storage**: 40 GB free disk space (100 GB+ recommended for production)
- **Network**: Two network interfaces (one for management, one for external/provider network)

### Software Requirements

- Ubuntu 22.04 LTS (Jammy Jellyfish) or similar
- Python 3.8 or higher
- Docker
- Ansible 2.14 or higher

## Environment Preparation

### 1. Update System Packages

```bash
sudo apt update && sudo apt upgrade -y
```

### 2. Install Required Dependencies

```bash
sudo apt install -y python3-dev libffi-dev gcc libssl-dev python3-pip python3-venv git
```

### 3. Configure Networking

Ensure your network interfaces are properly configured. Identify your network interfaces:

```bash
ip a
```

Edit the netplan configuration (typically at `/etc/netplan/00-installer-config.yaml`):

```yaml
network:
  version: 2
  ethernets:
    enp1s0:  # Management network interface
      dhcp4: true
    enp2s0:  # External/provider network interface
      dhcp4: false
```

Apply the network configuration:

```bash
sudo netplan apply
```

### 4. Disable Firewall (Optional, for testing)

```bash
sudo ufw disable
```

**Note**: For production environments, configure firewall rules appropriately instead of disabling it.

### 5. Install Docker

```bash
# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Add current user to docker group
sudo usermod -aG docker $USER

# Start and enable Docker service
sudo systemctl start docker
sudo systemctl enable docker
```

Log out and log back in for the group changes to take effect, or run:

```bash
newgrp docker
```

### 6. Verify Docker Installation

```bash
docker --version
docker ps
```

## Kolla-Ansible Installation

### 1. Create a Python Virtual Environment

```bash
python3 -m venv ~/kolla-venv
source ~/kolla-venv/bin/activate
```

**Important**: Always activate this virtual environment before running Kolla-Ansible commands.

### 2. Install Ansible and Kolla-Ansible

```bash
pip install -U pip
pip install 'ansible-core>=2.14,<2.16'
pip install git+https://opendev.org/openstack/kolla-ansible@stable/2024.1
```

### 3. Create Kolla Configuration Directory

```bash
sudo mkdir -p /etc/kolla
sudo chown $USER:$USER /etc/kolla
```

### 4. Copy Configuration Files

```bash
cp -r ~/kolla-venv/share/kolla-ansible/etc_examples/kolla/* /etc/kolla/
cp ~/kolla-venv/share/kolla-ansible/ansible/inventory/all-in-one ~/inventory
```

### 5. Install Ansible Galaxy Requirements

```bash
kolla-ansible install-deps
```

## Configuration

### 1. Generate Passwords

Kolla-Ansible uses various passwords for different services. Generate them automatically:

```bash
kolla-genpwd
```

This creates the file `/etc/kolla/passwords.yml` with randomly generated passwords.

### 2. Configure globals.yml

Edit the main configuration file:

```bash
nano /etc/kolla/globals.yml
```

Key configuration parameters to set:

```yaml
---
# Kolla options
kolla_base_distro: "ubuntu"
kolla_install_type: "source"

# OpenStack release
openstack_release: "2024.1"

# Network interface configuration
network_interface: "enp1s0"  # Management network interface
neutron_external_interface: "enp2s0"  # External/provider network interface

# Networking options
enable_haproxy: "no"  # Disable HAProxy for single node

# Virtual IP address (not used in single node, but required)
kolla_internal_vip_address: "10.0.2.15"  # Use your management IP

# Docker options
docker_registry: "quay.io"
docker_namespace: "openstack.kolla"

# Enable additional services (optional)
enable_cinder: "yes"
enable_cinder_backup: "no"
enable_heat: "yes"
enable_horizon: "yes"
enable_neutron_provider_networks: "yes"

# Neutron options
neutron_plugin_agent: "openvswitch"
```

**Important**: Replace interface names and IP addresses with values matching your environment.

### 3. Configure Inventory

The inventory file specifies which services run on which hosts. For a single-node deployment, the `all-in-one` inventory file is sufficient. Verify it:

```bash
cat ~/inventory
```

It should contain your host details, typically `localhost` or your hostname.

## Deployment

### 1. Bootstrap Servers

Prepare the host for deployment:

```bash
kolla-ansible -i ~/inventory bootstrap-servers
```

This step:
- Installs required packages
- Configures Docker
- Sets up directories

### 2. Perform Pre-deployment Checks

```bash
kolla-ansible -i ~/inventory prechecks
```

This validates that your environment meets all requirements. Fix any reported issues before proceeding.

### 3. Pull Docker Images

Download all required OpenStack service container images:

```bash
kolla-ansible -i ~/inventory pull
```

**Note**: This step can take 30-60 minutes depending on your internet connection, as it downloads several GB of container images.

### 4. Deploy OpenStack

Deploy all OpenStack services:

```bash
kolla-ansible -i ~/inventory deploy
```

This is the main deployment step and typically takes 20-40 minutes. Monitor the output for any errors.

### 5. Post-deployment Configuration

Generate the OpenStack admin credentials file:

```bash
kolla-ansible -i ~/inventory post-deploy
```

This creates the `/etc/kolla/admin-openrc.sh` file containing authentication credentials.

## Post-Deployment Verification

### 1. Source Admin Credentials

```bash
source /etc/kolla/admin-openrc.sh
```

### 2. Install OpenStack CLI Client

```bash
pip install python-openstackclient python-neutronclient python-cinderclient
```

### 3. Verify OpenStack Services

Check that all services are running:

```bash
openstack service list
```

List compute services:

```bash
openstack compute service list
```

Check network agents:

```bash
openstack network agent list
```

### 4. Access Horizon Dashboard

The OpenStack web interface (Horizon) should be accessible at:

```
http://<your-server-ip>
```

Login credentials:
- **Username**: admin
- **Password**: Found in `/etc/kolla/passwords.yml` (look for `keystone_admin_password`)

To retrieve the admin password:

```bash
grep keystone_admin_password /etc/kolla/passwords.yml
```

### 5. Create Test Resources

Create a simple test to verify functionality:

```bash
# Create a test network
openstack network create test-network

# Create a subnet
openstack subnet create --network test-network --subnet-range 192.168.100.0/24 test-subnet

# List images (should show Cirros or other images if configured)
openstack image list

# List flavors
openstack flavor list
```

## Troubleshooting

### Common Issues and Solutions

#### Issue: Pre-checks fail with network interface errors

**Solution**: Verify that the network interfaces specified in `globals.yml` exist and are correctly named:

```bash
ip a
```

Update `/etc/kolla/globals.yml` with the correct interface names.

#### Issue: Docker service not running

**Solution**: Start the Docker service:

```bash
sudo systemctl start docker
sudo systemctl enable docker
```

#### Issue: Container deployment fails

**Solution**: Check Docker logs:

```bash
docker ps -a
docker logs <container-id>
```

Review Kolla logs:

```bash
sudo journalctl -u docker
```

#### Issue: Cannot access Horizon dashboard

**Solution**: 
1. Verify Horizon container is running:
   ```bash
   docker ps | grep horizon
   ```

2. Check HAProxy configuration (if enabled):
   ```bash
   docker logs haproxy
   ```

3. Verify firewall settings and ensure port 80/443 are accessible.

#### Issue: Deployment hangs or times out

**Solution**:
1. Check system resources (CPU, RAM, disk space):
   ```bash
   df -h
   free -h
   top
   ```

2. Increase timeout values in `/etc/kolla/globals.yml`:
   ```yaml
   docker_api_timeout: 120
   ```

### Checking Service Status

Check the status of all Kolla containers:

```bash
docker ps -a
```

View logs for a specific service:

```bash
docker logs <container-name>
```

Restart a specific service:

```bash
docker restart <container-name>
```

## Useful Commands

### Reconfigure Services

After making configuration changes:

```bash
kolla-ansible -i ~/inventory reconfigure
```

### Upgrade OpenStack

To upgrade to a newer OpenStack release:

```bash
# Update kolla-ansible
pip install --upgrade git+https://opendev.org/openstack/kolla-ansible@stable/<new-release>

# Update configuration
nano /etc/kolla/globals.yml  # Update openstack_release

# Perform upgrade
kolla-ansible -i ~/inventory upgrade
```

### Destroy Deployment

To completely remove the OpenStack deployment:

```bash
kolla-ansible -i ~/inventory destroy --yes-i-really-really-mean-it
```

**Warning**: This will delete all data and cannot be undone.

### View All Passwords

```bash
cat /etc/kolla/passwords.yml
```

### Check Container Status

```bash
# List all containers
docker ps -a

# Check specific service
docker ps | grep <service-name>

# View container logs
docker logs -f <container-name>
```

### OpenStack CLI Commands

```bash
# Source credentials
source /etc/kolla/admin-openrc.sh

# Service operations
openstack service list
openstack endpoint list
openstack catalog list

# Compute operations
openstack server list
openstack flavor list
openstack image list

# Network operations
openstack network list
openstack subnet list
openstack router list
openstack security group list

# Volume operations (if Cinder enabled)
openstack volume list
openstack volume type list
```

## Additional Resources

- [Official Kolla-Ansible Documentation](https://docs.openstack.org/kolla-ansible/latest/)
- [OpenStack Documentation](https://docs.openstack.org/)
- [Kolla-Ansible GitHub Repository](https://github.com/openstack/kolla-ansible)
- [OpenStack CLI Reference](https://docs.openstack.org/python-openstackclient/latest/)

## Contributing

Contributions to improve this guide are welcome! Please feel free to submit issues or pull requests.

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Support

For issues related to:
- **Kolla-Ansible**: Check the [official documentation](https://docs.openstack.org/kolla-ansible/latest/) or [IRC channel](https://wiki.openstack.org/wiki/IRC)
- **This guide**: Open an issue in this repository

---

**Note**: This guide is intended for educational and testing purposes. For production deployments, additional security hardening, backup strategies, and high-availability configurations should be considered.