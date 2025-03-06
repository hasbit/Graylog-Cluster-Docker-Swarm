# **Deployment Guide - High-Availability Docker Swarm Cluster with Graylog and Traefik**

Starting Graylog in your Lab with cluster mode (docker swarm)

This guide will help you run Graylog in cluster mode on multiple nodes thanks to Docker Swarm !
You need to pay attention to all the steps to take before running the docker stack YML file, because they will help you to achieve a real cluster environment with high availability.

![graylog-cluster drawio](https://github.com/user-attachments/assets/f18e8109-d3e7-4b78-b5b5-882f750fbb15)


## **1. Hardware and Software Requirements**

### Hardware
- **3 Swarm Manager VMs**: `gl-swarm-01`, `gl-swarm-02`, `gl-swarm-03`

#### Example hardware Vmware

1. Create 3 VMs
   ![image](https://github.com/user-attachments/assets/c0b2d98c-645e-4433-95ec-f23e7fb94217)




2. Choose Host for the CPU on Hardware settings:

![image](https://github.com/user-attachments/assets/c0f2d6b3-ac28-4de9-a981-31030c5e0f9d)


1)

If not, you will have a message error for MongoDB: `WARNING: MongoDB 5.0+ requires a CPU with AVX support, and your current system does not appear to have that!`

3. Set VM MAX COUNT on the 3 VMs

To avoid errors on opensearch, set the max virtual memory areas to 262144
```
echo 'vm.max_map_count = 262144' | sudo tee -a /etc/sysctl.conf
```

After reboot, you can verify that the setting is still correct by running `sysctl vm.max_map_count`

### Software
- **OS**: Ubuntu 24.04
- **Docker**: Version 27.3.1
- **Traefik**: Reverse Proxy v3.2.1
- **Graylog**: Version 6.1.4
- **MongoDB**: Version 7.0.14
- **OpenSearch**: Version 2.15.0

---

## **2. VM and Network Configuration**

### 2.1 Configure Hosts

Edit and append DNS entries to the `/etc/hosts` file on **all VMs**:

```bash
cat <<EOF >> /etc/hosts
10.1.30.10   gl-swarm-01
10.1.30.11   gl-swarm-02
10.1.30.12   gl-swarm-03
10.1.30.100  graylog
EOF
```

### 2.2 Install Required Packages

Install the necessary tools on each **VM**:

```bash
# Update packages
sudo apt update && apt upgrade -y

# Install Docker and Docker Compose
 curl -fsSL https://get.docker.com -o get-docker.sh
 sudo sh get-docker.sh

sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker $USER
```

- **GlusterFS**: High-availability storage servers.
- **Keepalived**: For managing the **Virtual IP (VIP)**.
```
# Update package lists
sudo apt update -y

# Install GlusterFS repository
sudo add-apt-repository ppa:gluster/glusterfs-10 -y
sudo apt update -y

# Install GlusterFS server, Keepalived, wget, and curl
sudo apt install glusterfs-server keepalived wget curl -y

# Enable and start GlusterFS service
sudo systemctl enable glusterd --now

# Enable Keepalived (it needs to be configured before starting)
sudo systemctl enable keepalived

Do not start now the Keepalive service.

# Allow Docker Swarm communication ports
sudo iptables -A INPUT -p tcp --dport 2377 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 7946 -j ACCEPT
sudo iptables -A INPUT -p udp --dport 7946 -j ACCEPT
sudo iptables -A INPUT -p udp --dport 4789 -j ACCEPT

# Allow GlusterFS service
sudo iptables -A INPUT -p tcp --dport 24007 -j ACCEPT  # GlusterFS management
sudo iptables -A INPUT -p tcp --dport 24008 -j ACCEPT  # GlusterFS RDMA
sudo iptables -A INPUT -p tcp --dport 49152:49160 -j ACCEPT  # GlusterFS brick ports

# Allow Elasticsearch ports
sudo iptables -A INPUT -p tcp --dport 9300 -j ACCEPT  # Cluster communication
sudo iptables -A INPUT -p tcp --dport 9200 -j ACCEPT  # REST API

# Save iptables rules to persist after reboot
sudo apt install iptables-persistent -y
sudo netfilter-persistent save
sudo netfilter-persistent reload

```
Or simply disable it: `systemctl disable firewalld && systemctl stop firewalld`

### 2.4 Set Up Docker Swarm Cluster

Initialize Docker Swarm on **gl-swarm-01**:
```bash
sudo docker swarm init --advertise-addr 10.1.30.10
docker swarm join-token manager
```
Copy the command described and paste it to **gl-swarm-02** and **gl-swarm-03**

Join **gl-swarm-02** and **gl-swarm-03** to the cluster:
```bash
sudo docker swarm join --token <SWARM_TOKEN> 10.1.30.10:2377
```
Verify the cluster:
```bash
sudo docker node ls
```
![image](https://github.com/user-attachments/assets/902bc4f3-0c7a-4055-b222-51d1735f2ba6)

```
```
All nodes are part of Docker swarm cluster ! Then let's set GlusterFS for cluster storage that will be used by all of our containers across the swarm cluster.

## **3. GlusterFS Configuration**

### 3.1 Create Shared Volumes

On **gl-swarm-01**, **gl-swarm-02**, **gl-swarm-03**:

1. Create the storage:
   ```bash
   sudo mkdir /srv/glusterfs
   ```

2. Configure GlusterFS on **gl-swarm-01**:
   ```bash
   sudo gluster peer probe gl-swarm-02
   sudo gluster peer probe gl-swarm-03
   sudo gluster volume create gv0 replica 3 transport tcp gl-swarm-01:/srv/glusterfs gl-swarm-02:/srv/glusterfs gl-swarm-03:/srv/glusterfs force
   sudo gluster volume start gv0

   ![image](https://github.com/user-attachments/assets/439c9325-7f23-4260-9477-7242e76e9241)

   ```
   Verify gluster cluster with: `sudo gluster peer status`
   ![image](https://github.com/user-attachments/assets/04b85a5b-0bcd-467b-896d-c02fc6c21ded)



    


We will then use`/home/admin/mnt-glusterfs` as a mountpoint.

```
mkdir /home/admin/mnt-glusterfs
```

3. Mount the GlusterFS volume on **gl-swarm-01**:
   ```bash
   echo 'gl-swarm-01:/gv0    /home/admin/mnt-glusterfs    glusterfs    defaults,_netdev  0 0' | sudo tee -a /etc/fstab
   sudo systemctl daemon-reload && sudo mount -a
   { crontab -l; echo "@reboot mount -a"; } | sudo crontab -
   ```

   
5. Mount the GlusterFS volume on **gl-swarm-02**:
   ```bash
   echo 'gl-swarm-02:/gv0    /home/admin/mnt-glusterfs    glusterfs    defaults,_netdev  0 0' | sudo tee -a /etc/fstab
   sudo systemctl daemon-reload && sudo mount -a
   { crontab -l; echo "@reboot mount -a"; } | sudo crontab -
   ```
   
6. Mount the GlusterFS volume on **gl-swarm-03**:
   ```bash
   echo 'gl-swarm-03:/gv0    /home/admin/mnt-glusterfs    glusterfs    defaults,_netdev  0 0' | sudo tee -a /etc/fstab
   sudo systemctl daemon-reload && sudo mount -a
   { crontab -l; echo "@reboot mount -a"; } | sudo crontab -
   ```

 8.  Change the permissions according to your user, (mine is admin):
 ```
 sudo chown -R admin:admin /home/admin/mnt-glusterfs/
 ```

## **4. Keepalived Configuration (VIP)**

Create a `/etc/keepalived/keepalived.conf` file on each manager VM. 
Check your active network card before pasting the cat EOF command: `ip a`

For **gl-swarm-01**:
```bash
sudo bash -c 'cat <<EOF > /etc/keepalived/keepalived.conf
vrrp_instance VI_1 {
    state MASTER
    interface ens18  
    virtual_router_id 51
    priority 100      # Master node higher priority
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass somepassword
    }
    virtual_ipaddress {
        10.1.30.100/24  # VIP
    }
}
EOF'
```

Start/Restart Keepalived:
```bash
sudo systemctl start keepalived
sudo systemctl restart keepalived
```

For **gl-swarm-02**:
```bash
sudo bash -c 'cat <<EOF > /etc/keepalived/keepalived.conf
vrrp_instance VI_1 {
    state BACKUP
    interface ens18   # Network card (vérifiez with "ip a")
    virtual_router_id 51
    priority 90      # Master node higher priority
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass somepassword
    }
    virtual_ipaddress {
        10.1.30.100/24  # VIP
    }
}
EOF'
```

Restart Keepalived:
```bash
sudo systemctl start keepalived
sudo systemctl restart keepalived
```

For **gl-swarm-03**:
```bash
sudo bash -c 'cat <<EOF > /etc/keepalived/keepalived.conf
vrrp_instance VI_1 {
    state BACKUP
    interface ens18   # Network card (vérifiez with "ip a")
    virtual_router_id 51
    priority 80      # Master node higher priority
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass somepassword
    }
    virtual_ipaddress {
        10.1.30.100/24  # VIP
    }
}
EOF'
```

Restart Keepalived:
```bash
sudo systemctl start keepalived
sudo systemctl restart keepalived
```

## **5. Deploy the Stack**

### 5.1 Prepare network

On one of the nodes:

Create an overlay network: `docker network create -d overlay --attachable gl-swarm-net`, it will be used in the docker compose files as an external network. 
This network will allow to all containers across the nodes to communicate between them.

### 5.2 Prepare the containers folders

```
mkdir -p /home/admin/mnt-glusterfs/{graylog/{csv,gl01-data,gl02-data,gl03-data},opensearch/{os01-data,os02-data,os03-data},mongodb/{mongo01-data,mongo02-data,mongo03-data,initdb.d},traefik/certs}
```

The folder tree will look like this
```
/home/admin/mnt-glusterfs/
├── graylog
│   ├── csv
│   ├── gl01-data
│   ├── gl02-data
│   └── gl03-data
├── opensearch
│   ├── os01-data
│   ├── os02-data
│   └── os03-data
├── mongodb
│   ├── mongo01-data
│   ├── mongo02-data
│   ├── mongo03-data
│   └── initdb.d
└── traefik
    └── certs
```

### 5.2 Prepare the containers files

- Init script for set replicas mongodb
```
wget -O /home/admin/mnt-glusterfs/mongodb/init-replset.js https://github.com/hasbit/Graylog-Cluster-Docker-Swarm/blob/main/mnt-glusterfs/mongodb/init-replset.js
wget -O /home/admin/mnt-glusterfs/mongodb/initdb.d/init-replset.sh https://github.com/hasbit/Graylog-Cluster-Docker-Swarm/blob/main/mnt-glusterfs/mongodb/initdb.d/init-replset.sh
sudo chown -R 1000:1000 /home/admin/mnt-glusterfs/mongodb/init-replset.js
sudo chown -R 1000:1000 /home/admin/mnt-glusterfs/mongodb/initdb.d/
```

Chown 1000 is to set the ownership ID similar to inside the container, otherwise the replicaset initialization will not work.

- Traefik files and demo cert:
```
wget -O /home/admin/mnt-glusterfs/traefik/traefik.yaml https://github.com/hasbit/Graylog-Cluster-Docker-Swarm/blob/main/mnt-glusterfs/traefik/traefik.yaml


### Create Certificate with OpenSSL
openssl genpkey -algorithm RSA -out server.key -aes256
openssl req -new -key server.key -out server.csr
openssl x509 -req -days 365 -in server.csr -signkey server.key -out server.crt



```

- Docker stack compose file
```
wget -O /home/admin/docker-stack.yml https://github.com/hasbit/Graylog-Cluster-Docker-Swarm/blob/main/docker-stack-with-Traefik.yml
```

BE CAREFUL HERE ! Before running the stack read this below:

- The Docker configuration for deploying Graylog is defined in a single YAML file. However, when the containers are deployed, the volumes specified in the file point to local paths on the node where the container is running. If these paths are not shared across all nodes in the cluster, it will result in issues with data access or consistency.

- Without Keepalive, one problem remains, even if traefik is in swarm mode, as the DNS entry point to the IP Addresse of the first VM, if this VM is down, access to graylog will not be working. We need that the DNS point to the VIP so that any Traefik can respond.


### 5.3 Run the cluster !

```
docker stack deploy -c docker-stack.yml Graylog-Swarm
```
#### 5.3.1 Verify stack services 

To view if the service stack is opearationnel and everything has a replicas, run: `docker stack services Graylog-Swarm`

![image](https://github.com/user-attachments/assets/d37e3e52-9cc8-4ca0-9ab3-8fe0a2b68d42)




```
```
#### 5.3.1 Verify Graylog API

Check cluster node via API, use the HTTPS: `curl -u admin:admin -k https://graylog:443/api/system/cluster/nodes | jq .`


![image](https://github.com/user-attachments/assets/2783910b-219d-4e1a-b64e-f94743926a76)

```
```
## 6 DEFAULTS CREDS

- Graylog WEB UI
   - user: admin
   - pasword: admin

# 7 :blue_book: Docker-stack.YML config

[Docker Stack File Config] (https://github.com/hasbit/Graylog-Cluster-Docker-Swarm/blob/master/DockerStack-configfile.md)


# 8 Credits 


Thanks to https://github.com/s0p4L1n3 for the project

- https://workingtitle.pro/posts/graylog-on-docker-part-1/
- https://workingtitle.pro/posts/graylog-on-docker-part-2/
- https://workingtitle.pro/posts/graylog-on-docker-part-3/
- https://community.graylog.org/t/quick-guide-graylog-deployment-with-ansible-and-docker-swarm/12365
