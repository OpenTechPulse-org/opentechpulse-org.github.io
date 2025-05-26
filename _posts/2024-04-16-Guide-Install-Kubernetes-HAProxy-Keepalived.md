---
layout: post
title: Installing Kubernetes on Proxmox with HAProxy and Keepalived (Guide)
date: 2024-04-16 12:00:00 +/-TTTT
#categories: [Guides, Kubernetes]
tags: [Guides, kubernetes, haproxy, keepalived]     # TAG names should always be lowercase
#author: matija
#img_path: /assets/img/Guide-Kubernetes-haproxy-keepalived/
image: '/images/Guide-Kubernetes-haproxy-keepalived/Preview.webp'
toc: true
---

In this comprehensive guide, I will walk you through the process of setting up and configuring Kubernetes nodes equipped with HAProxy/Keepalived to ensure high availability. We will use Proxmox as our virtualization platform, a popular choice for deploying and managing virtual environments. The tutorial will cover the initial setup, including the installation of necessary software and configuration of network settings, followed by a step-by-step explanation of how to integrate HAProxy/Keepalived with Kubernetes. This setup will enhance your cluster's resilience to hardware and software failures, ensuring a robust and reliable system. Whether you are new to Kubernetes or looking to expand your knowledge, this guide will provide valuable insights and practical steps to help you build a high-availability Kubernetes cluster using Proxmox.

## **Prerequisites**

For this guide, I assume you have the following configuration:

| Nodes | Remarks |
| ----------- | ----------- |
| 3x VMs HAProxy / Keepalived | IPs: 192.168.1.10-12 |
| 3x VMs Kubernetes Controller Nodes | IPs: 192.168.1.15-17 | 
| x VMs Kubernetes Worker Nodes | IPs: 192.168.1.20-... | 

**VIP Configuration:**

1 VIP IP: 192.168.1.5

VIP Explanation:
: This is a Virtual IP address that can 'float' between hosts using Keepalived, allowing HAProxy to bind to it for high availability.

{: .warning }
Warning: The VIP IP must be free and not used elsewhere.

**Operating System:**

All nodes are using Ubuntu 22.04.4 LTS Server.

## **HAProxy / Keepalived nodes**

### Initial Setup

{: .important }
Repeat these steps for every HAProxy/Keepalived node. Don't forget to change the IP address for every node. Also do a reboot of the nodes once you complete all the steps.

#### Basic install operating system

  To set up your HAProxy and Keepalived nodes for high availability, start by installing Ubuntu 22.04.4 LTS Server on each virtual machine managed through Proxmox VE. This step-by-step guide assumes you are familiar with basic Proxmox operations.

  Download the ISO: Visit the [official Ubuntu website](https://ubuntu.com/download/server) and download the ISO file for Ubuntu 22.04.4 LTS Server.

  Upload the ISO to Proxmox: Log into your Proxmox web interface, navigate to 'Storage' under the 'Datacenter' settings, select your storage, and upload the Ubuntu ISO.

##### Create a New Virtual Machine:

      Click on "Create VM" in the Proxmox web interface.
      Enter the necessary VM configuration details such as VM ID, name, and operating system.
      For the OS type, ensure you select 'Linux' and for Version, choose '6.x - 2.6 Kernel'.
      Allocate the recommended CPU, memory, and disk resources according to your environment's specifications.
      Under the CD/DVD tab, select the uploaded Ubuntu ISO file.

##### Install Ubuntu:

      Start the VM and open the console from the Proxmox interface.
      Proceed with the Ubuntu installation by following the on-screen instructions. Ensure you install the standard server utilities and configure at least one network interface. Also be sure to enable the SSH server.

  3. Network Configuration:

     Proxmox typically handles network bridging automatically, but you should verify the network configuration to ensure your VM is accessible according to your network design.

  4. System Update:

      After installation, ensure that your system is up-to-date with the latest security patches and software updates. SSH into your VM and run:
     ```bash
      sudo apt update && sudo apt upgrade -y
      ```
      {: .nolineno }

- #### Configure Static IP Address

  After completing the basic installation of Ubuntu, the next crucial step is to configure the network interface to use a static IP address. This ensures that the VM has a consistent IP address that does not change with reboots, which is vital for network reliability and easier management.

  1. Identify the Interface: First, identify the network interface that you wish to configure. You can list all available interfaces by executing:
     ```shell
      ip link show
      ```
      {: .nolineno }

  2. Edit Netplan Configuration:
      - Ubuntu uses Netplan for network configuration. Locate the Netplan configuration files in /etc/netplan/.
      - Edit the appropriate YAML configuration file (typically named 00-installer-config.yaml or similar). Here’s an example configuration for a static IP:
        ```shell
        nano /etc/netplan/00-installer-config.yaml
        ```
        {: .nolineno }
        ```yaml
        # This is the network config written by 'subiquity'
        network:
          ethernets:
            ens18: # Change this to your interface name
              addresses:
              - 192.168.1.10/24 # Don't forget to change the IP for the other nodes
              nameservers:
                addresses:
                - 8.8.8.8
                search: []
              routes:
              - to: default
                via: 192.168.1.1
          version: 2
        ```
        {: file="/etc/netplan/00-installer-config.yaml" }
  3. Apply the Netplan changes:
      ```shell
      sudo netplan apply
      ```
      {: .nolineno }


- #### Setting the correct timezone
  Properly configuring the timezone of your server is crucial for syncing activities and logging events accurately. To set the correct timezone on your Ubuntu server, you can use the dpkg-reconfigure tzdata command, which provides a straightforward graphical interface for selecting your timezone.

  ```bash
  sudo dpkg-reconfigure tzdata
  ```
  {: .nolineno }

- #### Extend LVM volume to use full disk
  When using LVM, Ubuntu does not allocate all available disk space by default to allow for flexible storage management, including easy volume extension, snapshot creation, and optimized filesystem performance.

  Extending an LVM volume to use 100% of available disk space can maximize storage utilization, ensuring no space is wasted and all available capacity is accessible for applications and data.

  With the following commands we can extend the LVM to 100% disk usage.

  ```shell
  lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
  ```
  {: .nolineno }

  This command is used to extend the logical volume. By specifying -l +100%FREE, you are instructing the system to use 100% of the unallocated space available in the volume group (ubuntu-vg) for the logical volume (ubuntu-lv).

  ```shell
  resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv
  ```
  {: .nolineno }  

  Once the logical volume is extended, the filesystem itself must be resized to utilize the additional space. resize2fs is a tool used for resizing ext2, ext3, or ext4 filesystems.

### Configuration of Keepalived

Keepalived is a routing software that provides high availability by managing failover and load balancing among servers, using the Virtual Router Redundancy Protocol (VRRP).

In this case we have 3 HAProxy/Keepalived nodes. One of them (192.168.1.10) will become the MASTER and the other two will we BACKUP.

- #### Install Keepalived

  You can install Keepalived on Ubuntu with the following command:
  ```bash
  sudo apt install keepalived
  ```
  {: .nolineno }

- #### Configure Keepalived
  The configuration of keepalived is kept in the file /etc/keepalived/keepalived.conf

  On the first node (MASTER / 192.168.1.10) we open this file:
  ```bash
  sudo nano /etc/keepalived/keepalived.conf
  ```
  {: .nolineno }
  You clear the contents of this file and replace it with:
  ```text
  global_defs {
    notification_email {
    }
    router_id LVS_HAPROXY1
    vrrp_skip_check_adv_addr
    vrrp_garp_interval 0
    vrrp_gna_interval 0
    enable_script_security
  }

  vrrp_script chk_haproxy {
     user root
     script "/usr/bin/killall -0 haproxy"
     interval 2
     weight 2
  }

  vrrp_instance haproxy-vip {
     state MASTER
     priority 103
     interface ens18
     virtual_router_id 60
     advert_int 1
     authentication {
        auth_type PASS
        auth_pass 1111
     }
     unicast_src_ip 192.168.1.10
     unicast_peer {
        192.168.1.11
        192.168.1.12
     }
     virtual_ipaddress {
        192.168.1.5/24
     }
     track_script {
        chk_haproxy
     }
  }
  ```
  {: file="/etc/keepalived/keepalived.conf" }  

   On the second node (BACKUP / 192.168.1.11) the file should look like this:
  ```text
  global_defs {
    notification_email {
    }
    router_id LVS_HAPROXY2
    vrrp_skip_check_adv_addr
    vrrp_garp_interval 0
    vrrp_gna_interval 0
    enable_script_security
  }

  vrrp_script chk_haproxy {
     user root
     script "/usr/bin/killall -0 haproxy"
     interval 2
     weight 2
  }

  vrrp_instance haproxy-vip {
     state BACKUP
     priority 102
     interface ens18
     virtual_router_id 60
     advert_int 1
     authentication {
        auth_type PASS
        auth_pass 1111
     }
     unicast_src_ip 192.168.1.11
     unicast_peer {
        192.168.1.10
        192.168.1.12
     }
     virtual_ipaddress {
        192.168.1.5/24
     }
     track_script {
        chk_haproxy
     }
  }
  ```
  {: file="/etc/keepalived/keepalived.conf" }

   And finally on the third node (BACKUP / 192.168.1.12) the file should look like this:
  ```text
  global_defs {
    notification_email {
    }
    router_id LVS_HAPROXY3
    vrrp_skip_check_adv_addr
    vrrp_garp_interval 0
    vrrp_gna_interval 0
    enable_script_security
  }

  vrrp_script chk_haproxy {
     user root
     script "/usr/bin/killall -0 haproxy"
     interval 2
     weight 2
  }

  vrrp_instance haproxy-vip {
     state BACKUP
     priority 101
     interface ens18
     virtual_router_id 60
     advert_int 1
     authentication {
        auth_type PASS
        auth_pass 1111
     }
     unicast_src_ip 192.168.1.12
     unicast_peer {
        192.168.1.10
        192.168.1.11
     }
     virtual_ipaddress {
        192.168.1.5/24
     }
     track_script {
        chk_haproxy
     }
  }
  ```
  {: file="/etc/keepalived/keepalived.conf" }

  Don't forget to restart the keepalived service on all 3 nodes:
  ```bash
  sudo systemctl restart keepalived
  ```
  {: .nolineno }      

  In this setup where we have three HAProxy/Keepalived nodes, while vrrp_script checks if HAProxy is actively running by probing its process, it does not monitor the status of the node itself. To ensure robustness against node failures, Keepalived employs the Virtual Router Redundancy Protocol (VRRP). This protocol involves each node sending periodic 'heartbeats' or advertisements. If a backup node stops receiving these signals from the master node, it interprets this as a failure and promptly takes over as the new master. This seamless failover mechanism maintains service availability by effectively monitoring both the application's operational state and the health of the node.

### Configuration of HAProxy

HAProxy is a reliable, high-performance load balancer and proxy server for TCP and HTTP-based applications, widely used to ensure efficient distribution of network traffic across multiple servers.

  > Repeat these steps for every HAProxy/Keepalived node.
  {: .prompt-info }

- #### Install HAProxy

  You can install HAProxy on Ubuntu with the following command:
  ```bash
  sudo apt install haproxy
  ```
  {: .nolineno }

- #### Enable Non-local bind for the VIP address  

  To allow HAProxy to bind to a virtual IP (VIP) that isn't assigned to an interface on the server at the moment HAProxy starts, you need to enable non-local binding on your Linux system. This is necessary because, by default, Linux will prevent an application from binding to an IP address that is not locally configured on an interface. Follow these steps:

  1. Create the file 99-haproxy.conf in the /etc/sysctl.d/ directory:
     ```bash
     sudo nano /etc/sysctl.d/99-haproxy.conf
     ```
     {: .nolineno }
  2. Add the following line to the file:
     ```text
     net.ipv4.ip_nonlocal_bind=1
     ```
     {: file="/etc/sysctl.d/99-haproxy.conf" }

     Save and close the file.
  3. Apply the changes by running:
     ```bash
     sudo sysctl --system
     ```
     {: .nolineno }
     This command will load all sysctl settings, including your new configuration.

  By setting net.ipv4.ip_nonlocal_bind to 1, you allow HAProxy (or any other application) to bind to an IP address that is not currently assigned to a device on the server. This is particularly useful in high-availability setups like those using Keepalived where the VIP can float between different nodes based on failover or load balancing decisions.

- #### Configure HAProxy

  We are now going to put the right config for HAProxy

  1. Open the file haproxy.cfg in the /etc/haproxy/ directory:
     ```bash
     sudo nano /etc/haproxy/haproxy.cfg
     ```
     {: .nolineno }
  2. Delete all current lines and add the following to the file:
     ```text
     global
        log /dev/log  local0 info
        chroot      /var/lib/haproxy
        pidfile     /var/run/haproxy.pid
        maxconn     4000
        user        haproxy
        group       haproxy
        daemon
        stats socket /var/lib/haproxy/stats

     defaults
        mode                    http
        log                     global
        option                  httplog
        option                  dontlognull
        option http-server-close
        option forwardfor       except 127.0.0.0/8
        option                  redispatch
        retries                 1
        timeout http-request    10s
        timeout queue           20s
        timeout connect         5s
        timeout client          20s
        timeout server          20s
        timeout http-keep-alive 10s
        timeout check           10s

     listen stats
        bind 192.168.1.5:9000 # This is our VIP address
        mode http
        stats enable
        stats uri /stats
        stats realm Haproxy\ Statistics
        stats auth admin:password  # Replace 'admin' and 'password' with your desired username and password

     frontend kube-apiserver
        bind 192.168.1.5:6443 # This is our VIP address
        mode tcp
        option tcplog
        default_backend kube-apiserver

     backend kube-apiserver
        option httpchk GET /healthz
        http-check expect status 200
        mode tcp
        option ssl-hello-chk
        balance roundrobin
        server k8s-controller1 192.168.1.15:6443 check # IP address of our 1st controller
        server k8s-controller2 192.168.1.16:6443 check # IP address of our 2nd controller
        server k8s-controller3 192.168.1.17:6443 check # IP address of our 3th controller
     ```
     {: file="/etc/haproxy.haproxt.cfg" }
     Save and close the file.
  3. Apply the changes by restarting HAProxy:
     ```bash
     sudo systemctl restart haproxy
     ```
     {: .nolineno }

- #### Access HAProxy Stats

  We have configured a stats page that is running on our VIP IP address in HAProxy that can be accessed with the following URL:
  http://192.168.1.5:9000/stats

## **Kubernetes nodes**

### Initial setup

> All the steps outlined in [HAProxy/Keepalived nodes - Initial setup](#initial-setup) must be performed on all Kubernetes nodes, along with additional required steps. Also do a reboot of the nodes once you complete all the steps.
{: .prompt-warning }

- #### Disable swap

  Disabling swap on Kubernetes nodes ensures optimal performance as it prevents disk-based memory extension, which can slow down applications due to increased I/O operations. It also aids Kubernetes in efficient resource allocation by maintaining clear visibility into actual memory usage, thereby avoiding scheduling complications. Essentially, disabling swap helps Kubernetes to manage system resources more predictably and effectively.
  ```bash
  sudo nano /etc/fstab
  ```
  {: .nolineno }
  Comment out the line with "swap.img" in it:
  ```text
  ...
  #/swap.img      none    swap    sw      0       0
  ...
  ```
  {: file="/etc/fstab" }
  Save and close the file.

- #### Enable IP Forwarding and Bridge Call

  In Kubernetes, enabling IP forwarding is necessary to allow nodes to forward network traffic to pods, facilitating communication across different network interfaces. Additionally, enabling bridge calls ensures that traffic routed through a bridge (like those used by containers) is properly handled by iptables, supporting network policies and pod communication effectively. These settings are key for maintaining robust networking within a Kubernetes cluster.

  First we open the file /etc/modules
  ```bash
  sudo nano /etc/modules
  ```
  {: .nolineno }
  And we add the following lines to it:
  ```text
  overlay
  br_netfilter
  ```
  {: file="/etc/modules" }
  In a Kubernetes setup, adding overlay to /etc/modules enables overlay filesystems necessary for managing containerized applications. Additionally, including br_netfilter ensures that bridged network traffic can be properly filtered, crucial for maintaining network policies and pod communication within the cluster.

  Now we add some parameters to the /etc/sysctl.conf. For this execute following commands:
  ```bash
  echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
  echo "net.bridge.bridge-nf-call-iptables=1" | sudo tee -a /etc/sysctl.conf
  ```
  {: .nolineno }
  By adding net.ipv4.ip_forward=1 to /etc/sysctl.conf, you enable IP forwarding, crucial for allowing the node to route traffic between different pods across networks. The command net.bridge.bridge-nf-call-iptables=1 ensures that traffic passing through the network bridge is processed by iptables, allowing for proper network segmentation and policy enforcement within Kubernetes. These settings are necessary to facilitate robust network operations and security in a Kubernetes cluster.

### Setting up Containerd

> Warning: Must be done on **all** nodes.
{: .prompt-warning }

In a Kubernetes setup, containerd serves as the container runtime that directly manages the lifecycle of containers within the cluster. It handles tasks such as running containers, pulling images, and managing storage and network configurations. This is critical for the functioning of Kubernetes, as it relies on a container runtime to execute the containers that run the applications.

- #### Add docker GPG key

  Adding the Docker GPG key is a crucial security step when setting up Docker / Containerd on a system. This key is used to verify the authenticity of the Docker packages downloaded from its repository, ensuring that they haven't been tampered with or altered. By importing this GPG key, your system can trust the signed packages received from Docker's repository, maintaining the integrity and security of your Docker installation.

  Execute the following commands to add the GPG key to your node:
  ```bash
  sudo install -m 0755 -d /etc/apt/keyrings
  curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
  sudo chmod a+r /etc/apt/keyrings/docker.gpg
  ```
  {: .nolineno }

- #### Add Docker repository

  Adding the Docker repository to your system's package sources enables you to install and update Docker / Containerd directly from Docker, Inc.

  To add the repository to your node execute following commands:
  ```bash
  echo "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
  sudo apt-get update
  ```
  {: .nolineno }

- #### Install containerd
  
  Containerd is a high-performance container runtime that manages the complete container lifecycle of its host system, from image transfer and storage to container execution and supervision.

  The installation is very simple with only one command:
  ```bash
  sudo apt install -y containerd.io
  ```
  {: .nolineno }

- #### Install CNI plugins

  The installation of CNI (Container Network Interface) plugins is a crucial step for configuring networking in container environments, such as Kubernetes. These plugins provide the necessary network connectivity for containers and enable the application of network policies at runtime.

  First check [Releases · containernetworking/plugins](https://github.com/containernetworking/plugins/releases) for the latest release.

  Then execute following commands to install (Change the version to the latest release):
  ```bash
  wget https://github.com/containernetworking/plugins/releases/download/v1.4.1/cni-plugins-linux-amd64-v1.4.1.tgz
  mkdir -p /opt/cni/bin
  tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.4.1.tgz
  ```
  {: .nolineno }

- #### Configuring the systemd cgroup driver

  Configuring the systemd cgroup driver in containerd is an important step for ensuring that container processes are properly managed by the system’s init system, systemd. This configuration aligns the container runtime's resource handling with systemd's, enabling more efficient management of system resources. 

  First we need to create the directory and generate the config file with following commands:
  ```bash
  sudo mkdir -p /etc/containerd
  sudo containerd config default | sudo tee /etc/containerd/config.toml
  ```
  {: .nolineno }

  To use the systemd cgroup driver in /etc/containerd/config.toml with runc open the config file:
  ```bash
  sudo nano /etc/containerd/config.toml
  ```
  {: .nolineno }

  Find a line that starts with sandbox_image = (This is somewhere in the beginning of the file) and change it like this:
  ```text
  ...
  sandbox_image = "registry.k8s.io/pause:3.9"
  ...
  ```
  {: file="/etc/containerd/config.toml"  }

  Change runc.options SystemCgroup to True:
  ```text
  ...
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
	  ...
	  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
	    SystemdCgroup = True
  ...
  ```
  {: file="/etc/containerd/config.toml"  }

  And as a last step restart containerd:
  ```bash
  sudo systemctl restart containerd
  ```
  {: .nolineno } 
  This ensures that all changes that have been made are applied.

### Install Kubernetes

> Warning: Must be done on **all** nodes.
{: .prompt-warning }

- #### Add Kubernetes repositories

  When you install these repositories you can install the required packages using "apt"

  First we add this repo with the following commands:
  ```bash
  sudo apt-get install -y apt-transport-https ca-certificates curl
  curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
  echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
  ```
  {: .nolineno }

  Now we can update the package list and install Kubernetes components with the following commands:
  ```bash
  sudo apt-get update
  sudo apt-get install -y kubelet kubeadm kubectl
  sudo apt-mark hold kubelet kubeadm kubectl
  ```
  {: .nolineno }

  Using apt-mark hold for Kubernetes packages is important to prevent unintentional updates that could disrupt the stability of your Kubernetes cluster. By marking the packages as "held," you ensure that they remain at their current versions and are not automatically upgraded when running apt-get upgrade or similar commands. This is crucial because Kubernetes components often require specific versions to maintain compatibility and stability within the cluster. Holding the packages helps prevent unexpected changes that could lead to compatibility issues or downtime in your Kubernetes environment.

- #### Initializing the controller nodes 

  1. Initialize 1st controller

     SSH into the first controller and login as root and go to the root home directory:
     ```bash
     sudo su
     cd
     ```
     {: .nolineno }
     Create the file kubeadm-config.yaml:
     ```bash
     nano kubeadm-config.yaml
     ```
     {: .nolineno }
     Copy-paste the following:
     ```yaml
     apiVersion: kubeadm.k8s.io/v1beta3
     kind: ClusterConfiguration
     kubernetesVersion: "1.29.1"                         # Specify your Kubernetes version here
     controlPlaneEndpoint: "192.168.1.5:6443"            # Your HA VIP IP
     networking:
         serviceSubnet: "10.96.0.0/16"                   # IPv4 service subnets
         podSubnet: "10.244.0.0/16"                      # IPv4 pod subnets
     apiServer:
         extraArgs:
            "service-cluster-ip-range": "10.96.0.0/16"
     controllerManager:
         extraArgs:
            "cluster-cidr": "10.244.0.0/16"
            "service-cluster-ip-range": "10.96.0.0/16"
            "node-cidr-mask-size-ipv4": "24"
     ---
     apiVersion: kubeadm.k8s.io/v1beta3
     kind: InitConfiguration
     localAPIEndpoint:
         advertiseAddress: "192.168.1.15"                # Local IP
     nodeRegistration:
         kubeletExtraArgs:
            "node-ip": "192.168.1.15"                    # Node specific IP (Local IP)
     ```
     {: file="/root/kubeadm-config.yaml" }
     Save and close the file.

     Now we are going to initialize the first controller with the kubeadm command:
     ```bash
     kubeadm init --config kubeadm-config.yaml --upload-certs
     ```
     {: .nolineno }
     This command initializes a Kubernetes control-plane node using the configuration defined in the specified YAML file (kubeadm-config.yaml). Additionally, it uploads the generated TLS certificates to a Kubernetes Secret in the cluster, facilitating secure communication between control-plane components and worker nodes.
     > You'll need the certificateKey, token, and caCertHashes generated during the execution of kubeadm init --config kubeadm-config.yaml --upload-certs for configuring and authenticating additional control-plane nodes and worker nodes in your Kubernetes cluster. I advise you to copy them for quick access. 
     {: .prompt-tip }

  2. Initialize 2nd controller

     SSH into the second controller and login as root and go to the root home directory:
     ```bash
     sudo su
     cd
     ```
     {: .nolineno }
     Create the file kubeadm-config.yaml:
     ```bash
     nano kubeadm-config.yaml
     ```
     {: .nolineno }
     Copy-paste the following:
     ```yaml
     apiVersion: kubeadm.k8s.io/v1beta3
     kind: ClusterConfiguration
     kubernetesVersion: "1.29.1"                         # Specify your Kubernetes version here
     controlPlaneEndpoint: "192.168.1.5:6443"            # Your HA VIP IP
     networking:
         serviceSubnet: "10.96.0.0/16"                   # IPv4 service subnets
         podSubnet: "10.244.0.0/16"                      # IPv4 pod subnets
     apiServer:
         extraArgs:
            "service-cluster-ip-range": "10.96.0.0/16"
     controllerManager:
         extraArgs:
            "cluster-cidr": "10.244.0.0/16"
            "service-cluster-ip-range": "10.96.0.0/16"
            "node-cidr-mask-size-ipv4": "24"
     ---
     apiVersion: kubeadm.k8s.io/v1beta3
     kind: JoinConfiguration
     controlPlane:
      localAPIEndpoint:
         advertiseAddress: "192.168.1.16"                # Local IP
         bindPort: 6443
      certificateKey: "xxx"                              # The certificate key from the first control plane node
     discovery:
      bootstrapToken:
         apiServerEndpoint: "192.168.1.5"                # HA VIP IP
         token: "xxx"                                    # The token from the first controller
         caCertHashes:
            - "sha256:xxx"                               # The cert hash from the first controller
     nodeRegistration:
      kubeletExtraArgs:
         node-ip: "192.168.1.16"                         # Node specific IP (Local IP)
     ```
     {: file="/root/kubeadm-config.yaml" }
     Save and close the file.

     Now we are going to join this second controller with the following command:
     ```bash
     kubeadm join --config kubeadm-config.yaml
     ```
     {: .nolineno }
     This command is used to join additional control-plane nodes or worker nodes to an existing Kubernetes cluster.

  3. Initialize 3th controller

     SSH into the thirth controller and login as root and go to the root home directory:
     ```bash
     sudo su
     cd
     ```
     {: .nolineno }
     Create the file kubeadm-config.yaml:
     ```bash
     nano kubeadm-config.yaml
     ```
     {: .nolineno }
     Copy-paste the following:
     ```yaml
     apiVersion: kubeadm.k8s.io/v1beta3
     kind: ClusterConfiguration
     kubernetesVersion: "1.29.1"                         # Specify your Kubernetes version here
     controlPlaneEndpoint: "192.168.1.5:6443"            # Your HA VIP IP
     networking:
         serviceSubnet: "10.96.0.0/16"                   # IPv4 service subnets
         podSubnet: "10.244.0.0/16"                      # IPv4 pod subnets
     apiServer:
         extraArgs:
            "service-cluster-ip-range": "10.96.0.0/16"
     controllerManager:
         extraArgs:
            "cluster-cidr": "10.244.0.0/16"
            "service-cluster-ip-range": "10.96.0.0/16"
            "node-cidr-mask-size-ipv4": "24"
     ---
     apiVersion: kubeadm.k8s.io/v1beta3
     kind: JoinConfiguration
     controlPlane:
      localAPIEndpoint:
         advertiseAddress: "192.168.1.17"                # Local IP
         bindPort: 6443
      certificateKey: "xxx"                              # The certificate key from the first control plane node
     discovery:
      bootstrapToken:
         apiServerEndpoint: "192.168.1.5"                # HA VIP IP
         token: "xxx"                                    # The token from the first controller
         caCertHashes:
            - "sha256:xxx"                               # The cert hash from the first controller
     nodeRegistration:
      kubeletExtraArgs:
         node-ip: "192.168.1.17"                         # Node specific IP (Local IP)
     ```
     {: file="/root/kubeadm-config.yaml" }
     Save and close the file.

     Now we are going to join this thirth controller with the following command:
     ```bash
     kubeadm join --config kubeadm-config.yaml
     ```
     {: .nolineno }
     This command is used to join additional control-plane nodes or worker nodes to an existing Kubernetes cluster.

  4. Install Calico

     Calico is essential in Kubernetes to manage and secure network communication both within the cluster and between the cluster and external services. It provides network segmentation, which isolates different workloads from each other, enhancing security. Additionally, Calico enables enforcement of detailed network policies that dictate how pods communicate with each other, helping prevent unauthorized access and mitigating potential attacks within the cluster. This makes Calico a crucial tool for maintaining robust network security and performance in Kubernetes environments.

     You can install Calico directly by applying a YAML file from the official Calico site. This YAML file contains all the necessary Kubernetes resources (like DaemonSets, Deployments, ConfigMaps, etc.) needed for Calico.
     ```bash
     kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
     ```
     {: .nolineno }
     After applying the YAML file, check that all the Calico pods are running by using the following command:
     ```bash
     kubectl get pods -n kube-system | grep calico
     ```
     {: .nolineno }
     You should see the Calico pods listed as running. Something like this:
     ```bash
     root@k8s-master1:~# kubectl get pods -n kube-system | grep calico
     calico-kube-controllers-7ddc4f45bc-dbxdg   1/1     Running   5 (19d ago)    78d
     calico-node-4cdbk                          1/1     Running   4 (18d ago)    75d
     calico-node-4dmqk                          1/1     Running   1 (73d ago)    78d
     calico-node-5jlzz                          1/1     Running   2 (18d ago)    75d
     calico-node-6ks2f                          1/1     Running   2 (18d ago)    76d
     calico-node-6lms7                          1/1     Running   2 (73d ago)    75d
     calico-node-bkmws                          1/1     Running   2 (18d ago)    78d
     calico-node-ck774                          1/1     Running   1 (73d ago)    78d
     calico-node-cqw2p                          1/1     Running   2 (18d ago)    75d
     calico-node-fqhnl                          1/1     Running   3 (18d ago)    76d
     calico-node-frhvb                          1/1     Running   2 (18d ago)    76d
     calico-node-kht48                          1/1     Running   4 (18d ago)    75d
     calico-node-mlds7                          1/1     Running   1 (73d ago)    78d
     calico-node-q7kst                          1/1     Running   2 (18d ago)    75d
     calico-node-rbbsx                          1/1     Running   2 (18d ago)    76d
     calico-node-rdtjp                          1/1     Running   2 (18d ago)    76d
     calico-node-t9cjl                          1/1     Running   2 (18d ago)    76d
     calico-node-tq4nv                          1/1     Running   2 (18d ago)    76d
     ```
     {: .nolineno }

     When this is all good you could do a 'kubectl get nodes' and the output should be this:
     ```bash
     root@k8s-master1:~# kubectl get nodes
     NAME           STATUS   ROLES           AGE   VERSION
     k8s-master1    Ready    control-plane   78d   v1.29.1
     k8s-master2    Ready    control-plane   78d   v1.29.1
     k8s-master3    Ready    control-plane   78d   v1.29.1
     ```
     {: .nolineno }
     All the controllers should be status Ready.

- #### Joining the worker nodes

     Worker nodes in a Kubernetes cluster are the machines that run your applications and workloads. They are responsible for executing containerized tasks assigned by the control plane. Each worker node runs a kubelet, which is an agent that communicates with the Kubernetes master to ensure containers are running as intended. Worker nodes also manage networking for the containers, allocate resources, and handle storage, providing the necessary infrastructure to support the demands of your applications. Essentially, they do the heavy lifting that keeps applications running smoothly and efficiently.

     To join a worker node to the cluster you need to do the following.

     SSH into the worker node and login as root and go to the root home directory:
     ```bash
     sudo su
     cd
     ```
     {: .nolineno }
     Create the file kubeadm-config.yaml:
     ```bash
     nano kubeadm-config.yaml
     ```
     {: .nolineno }
     Copy-paste the following:
     ```yaml
     apiVersion: kubeadm.k8s.io/v1beta3
     kind: JoinConfiguration
     discovery:
      bootstrapToken:
         apiServerEndpoint: "192.168.1.5:6443"           # HA VIP IP
         token: "xxx"                                    # The token from the first controller
         caCertHashes:
            - "sha256:xxx"                               # The cert hash from the first controller
     nodeRegistration:
      kubeletExtraArgs:
         node-ip: "192.168.1.20"                         # Node specific IP (Local IP) / DON'T FORGET TO CHANGE
     ```
     {: file="/root/kubeadm-config.yaml" }
     Save and close the file.

     Now we are going to join this thirth controller with the following command:
     ```bash
     kubeadm join --config kubeadm-config.yaml
     ```
     {: .nolineno }
     This command is used to join additional worker nodes to an existing Kubernetes cluster.

     When you joined the last worker you can get a list of all nodes in the cluster by running following command on controller 1:
     ```bash
     root@k8s-master1:~# kubectl get nodes
     NAME           STATUS   ROLES           AGE   VERSION
     k8s-master1    Ready    control-plane   78d   v1.29.1
     k8s-master2    Ready    control-plane   78d   v1.29.1
     k8s-master3    Ready    control-plane   78d   v1.29.1
     k8s-storage1   Ready                    75d   v1.29.1
     k8s-storage2   Ready                    75d   v1.29.1
     k8s-storage3   Ready                    75d   v1.29.1
     k8s-storage4   Ready                    75d   v1.29.1
     k8s-worker1    Ready                    78d   v1.29.1
     k8s-worker10   Ready                    75d   v1.29.1
     k8s-worker2    Ready                    76d   v1.29.1
     k8s-worker3    Ready                    76d   v1.29.1
     k8s-worker4    Ready                    76d   v1.29.1
     k8s-worker5    Ready                    76d   v1.29.1
     k8s-worker6    Ready                    76d   v1.29.1
     k8s-worker7    Ready                    76d   v1.29.1
     k8s-worker8    Ready                    76d   v1.29.1
     k8s-worker9    Ready                    75d   v1.29.1
     ```
     {: .nolineno }
     You see that the ROLE is empy, we can define this by running following command on controller 1:
     ```bash
     kubectl label node k8s-worker1 node-role.kubernetes.io/worker=
     ```
     {: .nolineno }
     Do this for every worker node and you will get something like this:
     ```bash
     root@k8s-master1:~# kubectl get nodes
     NAME           STATUS   ROLES           AGE   VERSION
     k8s-master1    Ready    control-plane   78d   v1.29.1
     k8s-master2    Ready    control-plane   78d   v1.29.1
     k8s-master3    Ready    control-plane   78d   v1.29.1
     k8s-storage1   Ready    worker          75d   v1.29.1
     k8s-storage2   Ready    worker          75d   v1.29.1
     k8s-storage3   Ready    worker          75d   v1.29.1
     k8s-storage4   Ready    worker          75d   v1.29.1
     k8s-worker1    Ready    worker          78d   v1.29.1
     k8s-worker10   Ready    worker          75d   v1.29.1
     k8s-worker2    Ready    worker          76d   v1.29.1
     k8s-worker3    Ready    worker          76d   v1.29.1
     k8s-worker4    Ready    worker          76d   v1.29.1
     k8s-worker5    Ready    worker          76d   v1.29.1
     k8s-worker6    Ready    worker          76d   v1.29.1
     k8s-worker7    Ready    worker          76d   v1.29.1
     k8s-worker8    Ready    worker          76d   v1.29.1
     k8s-worker9    Ready    worker          75d   v1.29.1
     ```
     {: .nolineno }

## **Summary**

We have successfully set up a high-availability configuration using HAProxy and Keepalived, and established three Kubernetes controller nodes to ensure robust management and fault tolerance. Additionally, worker nodes have been integrated into the cluster, enhancing its capacity to handle workloads. With this robust infrastructure in place, we are now ready to deploy and manage applications effectively. It's time to put the system to the test and start deploying our services! 

Feel free to make suggestions or leave a comment if you have any questions or need further clarification.
