
---

## **Step-by-Step Guide for Deploying Production-Ready OpenStack with Kolla-Ansible**

---

### **1. Environment Preparation**

#### **1.1. Hardware and Network Requirements**

- **Controller Node(s)**: At least 2 nodes for high availability (can be more for scaling).
- **Compute Node(s)**: At least 2 nodes for load distribution.
- **Storage Node(s)**: If using Ceph for storage (one or more nodes).
- **Networking**: 3 types of networks—management, internal, and public.

**Recommended specs** (for each node):
- **Controller Node**:  
  - CPU: 4+ cores  
  - RAM: 16GB+  
  - Disk: 100GB+ (SSD recommended for faster I/O)
- **Compute Node**:  
  - CPU: 8+ cores  
  - RAM: 32GB+  
  - Disk: 100GB+ (SSD recommended)
- **Storage Node** (if using Ceph):  
  - CPU: 4 cores  
  - RAM: 16GB  
  - Disk: 1TB+ (SSDs or spinning disks depending on the workload)
- **Network**: Ensure network isolation between **management**, **internal**, and **external** networks.

#### **1.2. Operating System Setup**
- Use **Ubuntu 20.04 LTS** or **CentOS 8** for Kolla-Ansible compatibility.
  
  **On all nodes**:
  ```bash
  # Update OS and install necessary packages
  sudo apt update -y
  sudo apt upgrade -y
  sudo apt install -y python3-pip git sshpass net-tools curl
  sudo apt install -y software-properties-common python3-dev python3-setuptools
  ```

  **Install Docker and Docker Compose**:
  ```bash
  # Install Docker
  sudo apt install -y docker.io
  sudo systemctl enable --now docker

  # Install Docker Compose (optional but recommended)
  sudo apt install -y docker-compose
  ```

  **Install Ansible**:
  ```bash
  sudo apt install -y ansible
  ```

---

### **2. Install Kolla-Ansible**

#### **2.1. Clone Kolla-Ansible repository**
```bash
git clone https://opendev.org/openstack/kolla-ansible
cd kolla-ansible
```

#### **2.2. Install Python dependencies**
```bash
pip install -r requirements.txt
```

#### **2.3. Install Kolla-Ansible**
```bash
sudo pip install .
```

---

### **3. Configure Kolla-Ansible**

#### **3.1. Copy Kolla Configuration Files**
Copy Kolla's sample configuration files to `/etc/kolla/`:
```bash
sudo cp -r etc/kolla /etc/
sudo cp -r etc/ansible /etc/
```

#### **3.2. Configure Global Settings (`/etc/kolla/globals.yml`)**

Edit the `globals.yml` file to define key deployment settings:
```bash
sudo nano /etc/kolla/globals.yml
```

Set the following:
```yaml
kolla_base_distro: "ubuntu"  # or "centos" if using CentOS
kolla_install_type: "source"  # Use "binary" for a binary install
openstack_release: "yoga"     # Replace with your desired release (e.g., wallaby, xena)

# Networking configuration
network_interface: "eth0"
neutron_external_interface: "eth1"
neutron_network_type: "vxlan"  # Options: "vxlan", "vlan", "flat"
neutron_vlan_ranges: "physnet1:1000:2000"
neutron_bridge_mappings: "physnet1:br-ex"

# Enable services
enable_horizon: "yes"
enable_keystone: "yes"
enable_glance: "yes"
enable_nova: "yes"
enable_neutron: "yes"
enable_cinder: "yes"
enable_swift: "no"  # Optional (set to "yes" if using Object Storage)

# Container settings
docker_registry: "localhost:5000"   # Adjust according to your registry
kolla_enable_debug: "no"            # Set to "yes" for debug mode

# SSL for Horizon (optional but recommended)
horizon_ssl: "yes"
horizon_ssl_keyfile: "/etc/kolla/certificates/horizon.key"
horizon_ssl_certfile: "/etc/kolla/certificates/horizon.crt"
horizon_ssl_ca_certfile: "/etc/kolla/certificates/ca.crt"
```

#### **3.3. Configure Inventory File (`/etc/ansible/hosts`)**

Update the inventory file with the IP addresses and hostnames of your nodes:
```bash
sudo nano /etc/ansible/hosts
```

Example:
```ini
[controller]
controller1 ansible_host=192.168.1.10
controller2 ansible_host=192.168.1.11

[compute]
compute1 ansible_host=192.168.1.20
compute2 ansible_host=192.168.1.21

[storage]
controller1

[network]
controller1
```

- **controller**: Nodes running the control plane services (Keystone, Nova, Neutron, etc.).
- **compute**: Nodes running the compute services.
- **storage**: Storage nodes for Ceph (if applicable).
- **network**: Networking services, often on the controller nodes.

---

### **4. Pre-Deployment Checks**

Run Kolla’s pre-deployment checks to validate your environment:
```bash
kolla-ansible -i /etc/ansible/hosts pre-deploy
```

This step will ensure that all nodes are reachable, Docker is installed, and system dependencies are satisfied.

---

### **5. Deploy OpenStack**

Once the pre-deployment check is successful, you can begin the OpenStack deployment:

```bash
kolla-ansible -i /etc/ansible/hosts deploy
```

This will start deploying the various OpenStack services (Keystone, Nova, Glance, Neutron, etc.) as Docker containers. This process can take some time depending on the number of nodes and services being deployed.

---

### **6. Post-Deployment Steps**

After the deployment completes successfully, run the following to finalize and initialize services:

```bash
kolla-ansible -i /etc/ansible/hosts post-deploy
```

- This will:
  - Configure **Horizon** (dashboard) access.
  - Set up the **Keystone** service endpoints.
  - Check the OpenStack services for health.
  - Start and enable services (Keystone, Glance, Nova, Neutron, etc.).

---

### **7. Verify Deployment**

#### **7.1. Access Horizon (Dashboard)**

If you’ve enabled **Horizon**, you can access the OpenStack dashboard via a web browser:
```bash
https://<controller_ip>/dashboard
```
Login with the **admin** user and **admin** password (or as configured).

#### **7.2. Verify OpenStack Services**

Use the **openstack client** to verify if the services are running:
```bash
source /etc/kolla/admin-openrc.sh
openstack service list
```

Check the status of each OpenStack service to ensure they are up and running.

---

### **1. Enable HAProxy and Keepalived in Kolla-Ansible**

#### **1.1. Update `/etc/kolla/globals.yml` for HA Configuration**

Open the `globals.yml` file to enable HAProxy and Keepalived for high availability:

```bash
sudo nano /etc/kolla/globals.yml
```

Set the following options:
```yaml
# Enable HAProxy for load balancing services
enable_haproxy: "yes"

# Enable Keepalived for virtual IPs (VIPs) and automatic failover
enable_keepalived: "yes"

# Enable HA for various OpenStack services (e.g., Keystone, Nova)
enable_keystone: "yes"
enable_nova: "yes"
enable_neutron: "yes"

# Specify your management and external network interfaces
network_interface: "eth0"
neutron_external_interface: "eth1"

# Optionally configure SSL for Horizon
horizon_ssl: "yes"
horizon_ssl_keyfile: "/etc/kolla/certificates/horizon.key"
horizon_ssl_certfile: "/etc/kolla/certificates/horizon.crt"
horizon_ssl_ca_certfile: "/etc/kolla/certificates/ca.crt"
```

#### **1.2. Configure HAProxy Settings in `/etc/kolla/globals.yml`**

You can also configure more specific settings for **HAProxy** load balancing, depending on your environment. This could include specifying which services to load balance, the backend servers, etc.

Example:
```yaml
# Enable or disable the use of HAProxy for specific services
haproxy_keystone: "yes"  # Enable load balancing for Keystone
haproxy_nova: "yes"      # Enable load balancing for Nova
haproxy_glance: "yes"    # Enable load balancing for Glance
haproxy_neutron: "yes"   # Enable load balancing for Neutron

# Optional: Specify the interface for the internal and external traffic
haproxy_internal_interface: "eth0"  # Internal network interface for HAProxy
haproxy_external_interface: "eth1"  # External network interface for HAProxy
```

#### **1.3. Configure Keepalived VIP Settings**

You can specify the **virtual IPs** (VIPs) in the `globals.yml` file for **Keepalived** to ensure your OpenStack services are highly available across controller nodes.

Example:
```yaml
# Enable Keepalived for HA VIPs
enable_keepalived: "yes"

# Define the Virtual IP (VIP) for services like Keystone, Nova
keepalived_virtual_ip: "192.168.1.100"  # Example VIP for the external network

# Define the priority for each controller node in Keepalived
keepalived_controller1_priority: 101
keepalived_controller2_priority: 100
```

This will set up **Keepalived** to monitor the availability of services on the controller nodes and ensure the **VIP** (`192.168.1.100` in this example) always points to the active controller node.

---

### **2. Deploy OpenStack with HAProxy and Keepalived**

After modifying the `globals.yml` file, you're ready to deploy OpenStack using Kolla-Ansible, which will automatically set up **HAProxy** and **Keepalived** for high availability.

#### **2.1. Pre-deployment Checks**
Run the pre-deployment check to verify that all your settings and configurations are correct:

```bash
kolla-ansible -i /etc/ansible/hosts pre-deploy
```

This step will ensure that all necessary dependencies are in place and that the network and configuration settings are correct.

#### **2.2. Deploy OpenStack**
Deploy the OpenStack services:
```bash
kolla-ansible -i /etc/ansible/hosts deploy
```

Kolla-Ansible will deploy OpenStack as containers, while simultaneously configuring **HAProxy** for load balancing and **Keepalived** for high availability and failover.

---

### **3. Verify HAProxy and Keepalived Deployment**

#### **3.1. Verify HAProxy Load Balancing**
After deployment, verify that **HAProxy** is up and running, and check if it is properly load balancing the OpenStack services.

- **Check HAProxy status**:
  ```bash
  sudo systemctl status haproxy
  ```

- **Check HAProxy configuration**:
  Verify the HAProxy configuration for your services (Keystone, Nova, Glance, etc.):

  ```bash
  sudo nano /etc/kolla/haproxy/haproxy.cfg
  ```

  Ensure that the backend sections have been created for services like **Keystone** (for example):
  ```ini
  backend keystone_back
    server controller1 192.168.1.10:5000 check
    server controller2 192.168.1.11:5000 check
  ```

- **Test Load Balancing**:  
  Access your **Keystone** or **Nova** services (depending on what you've configured) via the load balancer’s VIP and verify that traffic is correctly distributed between your controller nodes.

  Example:
  ```bash
  curl -k https://192.168.1.100:5000/v3/  # Access Keystone via the VIP
  ```

#### **3.2. Verify Keepalived VIP Failover**
To test **Keepalived** failover, you can stop or restart the service on the current **master** controller and check if the **VIP** fails over to the other controller node.

- **Stop Keepalived on Controller1** (assuming Controller1 is the master):
  ```bash
  sudo systemctl stop keepalived
  ```

- **Check VIP on Controller2**:
  After stopping **Keepalived** on Controller1, verify that the VIP has moved to **Controller2**:
  ```bash
  ip addr show eth0
  ```

  The **VIP** should now appear on **Controller2**. This confirms that **Keepalived** is working for automatic failover.

---

### **4. Optional: Monitor HAProxy and Keepalived**

To keep an eye on the health of your OpenStack services, you can use **HAProxy stats** and **Keepalived logs**.

#### **4.1. HAProxy Stats Page**
Enable the **HAProxy stats page** in your configuration by adding the following to `/etc/kolla/haproxy/haproxy.cfg` (if not already enabled):

```ini
listen stats
   bind *:9000
   mode http
   stats uri /haproxy_stats
   stats auth admin:admin  # Set a password for accessing stats
```

Access the stats page via:
```
http://<haproxy-vip>:9000/haproxy_stats
```

#### **4.2. Keepalived Logs**
Monitor **Keepalived** logs to ensure it’s correctly handling VIP failovers. Check logs on both controller nodes:

```bash
tail -f /var/log/syslog | grep keepalived
```

---

### **5. Scaling and Additional Configuration**

As your cloud grows, you can scale services across additional controller and compute nodes. Kolla-Ansible allows you to easily add more nodes to your deployment by updating the inventory and rerunning the deployment commands.

- **Scale Compute Nodes**:  
  To add more compute nodes to your cloud, update the inventory file (`/etc/ansible/hosts`) with new compute node IP addresses and then run:
  ```bash
  kolla-ansible -i /etc/ansible/hosts deploy
  ```

- **Scale Services**:  
  You can also add services or scale existing ones by updating the `globals.yml` and `ansible/hosts` files and redeploying.

---

### **Conclusion**

By following these steps, **Kolla-Ansible** will automatically configure **HAProxy** and **Keepalived** for you, making your OpenStack deployment **production-ready** with high availability and load balancing. All that’s left for you to do is verify and monitor the deployment, ensuring that services are properly distributed across your controller nodes and that VIPs fail over seamlessly.

Feel free to reach out if you have further questions or need help with any part of the process!
