# Pi-hole in Kubernetes: Deployment on Raspberry Pi 5

> **Note:** For security and privacy reasons, all IP addresses, MAC addresses, and hostnames in this documentation have been changed from their actual values. The network configuration principles remain accurate, but the specific addresses shown are examples only.

## Project Overview

This repository documents my deployment of Pi-hole within a Kubernetes (MicroK8s) cluster on a Raspberry Pi 5 running Ubuntu 24.04 LTS. Pi-hole serves as a network-wide ad blocker and DHCP server, replacing my AT&T router’s DNS and DHCP functionality, all managed via Kubernetes for resilience and scalability.

## System Specifications

![Raspberry Pi 5 Hardware](images/physical-pi5.png)  
*My Raspberry Pi 5 setup*

- **Hardware**: Raspberry Pi 5 (8GB RAM)  
- **Storage**: 64GB microSD card  
- **Operating System**: Ubuntu 24.04 LTS (64-bit)  
- **Network**: Static IP configuration on Ethernet (eth0)  
- **Router**: AT&T Fiber (BGW320-505)  
- **Tools**: Docker (26.1.3), MicroK8s (v1.32.2)

## Installation Steps

### 1. Update System and Install Tools

I started by updating the system and installing necessary tools with the commands: sudo apt update && sudo apt upgrade -y, followed by sudo apt install docker.io -y. I added myself to the Docker group with sudo usermod -aG docker $USER && newgrp docker. Then, I installed MicroK8s using sudo snap install microk8s --classic and added myself to its group with sudo usermod -aG microk8s $USER && newgrp microk8s.

### 2. Configure MicroK8s

I enabled DNS and storage in MicroK8s with microk8s enable dns hostpath-storage, then verified the node status with microk8s kubectl get nodes, which showed "bryan-pi Ready <none> <age> v1.32.2".

### 3. Deploy Pi-hole in Kubernetes

I stopped the native Pi-hole service with sudo systemctl stop pihole-FTL, created persistent directories with mkdir -p /home/bryan/pihole/etc-pihole /home/bryan/pihole/etc-dnsmasq.d, and set permissions with chmod -R 755 /home/bryan/pihole. Then, I applied the Kubernetes deployment with microk8s kubectl apply -f pihole-deployment.yaml.

![Pi-hole Dashboard](images/pihole-dashboard.png)  
*Pi-hole admin interface showing statistics and status*

### 4. Post-Deployment

I secured the admin password by running microk8s kubectl exec -it pihole-<pod-name> -- pihole -a -p Coffee10!. I enabled DHCP in the Pi-hole dashboard with a range of 192.168.1.100-199 and disabled the router’s DHCP.

## Implementation Process

### 1. Network Configuration

I configured a static IP using Netplan by editing /etc/netplan/01-netcfg.yaml with the following content: network: version: 2, renderer: networkd, ethernets: eth0: match: macaddress: "2c:cf:67:23:cb:8e", dhcp4: no, addresses: - 192.168.1.200/24, nameservers: addresses: [8.8.8.8, 8.8.4.4], routes: - to: default, via: 192.168.1.254. I cleaned up conflicts by running sudo rm /etc/netplan/50-cloud-init.yaml, sudo chmod 600 /etc/netplan/01-netcfg.yaml, and sudo netplan apply.

### 2. Kubernetes Deployment

I defined pihole-deployment.yaml with this configuration: apiVersion: apps/v1, kind: Deployment, metadata: name: pihole, namespace: default, spec: replicas: 1, selector: matchLabels: app: pihole, template: metadata: labels: app: pihole, spec: hostNetwork: true, containers: - name: pihole, image: pihole/pihole:latest, env: - name: ServerIP, value: "192.168.1.200", - name: TZ, value: "America/Chicago", - name: WEBPASSWORD, value: "Coffee10!", - name: DHCP_ROUTER, value: "192.168.1.254", volumeMounts: - name: etc-pihole, mountPath: /etc/pihole, - name: etc-dnsmasq, mountPath: /etc/dnsmasq.d, ports: - containerPort: 53, protocol: UDP, - containerPort: 53, protocol: TCP, - containerPort: 67, protocol: UDP, - containerPort: 80, protocol: TCP, securityContext: capabilities: add: ["NET_ADMIN"], volumes: - name: etc-pihole, hostPath: path: /home/bryan/pihole/etc-pihole, type: Directory, - name: etc-dnsmasq, hostPath: path: /home/bryan/pihole/etc-dnsmasq.d, type: Directory.

### 3. System Monitoring & Optimization

I verified system health with sudo systemctl status docker, which showed "Active: active (running)", microk8s status, which confirmed "microk8s is running", and microk8s kubectl logs -l app=pihole, which logged FTL binding to ports 53, 80, and 443.

![System Health Statistics](images/system-data.png)  
*Terminal output showing pod status and logs*

## Troubleshooting Challenges & Solutions

### 1. DNS Timeouts

**Challenge**: nslookup google.com 192.168.1.200 timed out initially.  
**Solution**: I added hostNetwork: true to the YAML, restarted the pod, and verified DNS with nslookup google.com 192.168.1.200, which resolved to 142.251.35.238.

### 2. DHCP Handoff Conflicts

**Challenge**: Network outage occurred when disabling router DHCP first.  
**Solution**: I enabled Pi-hole DHCP first with a range of 192.168.1.100-199, then disabled router DHCP, and confirmed leases in the dashboard.

### 3. NET_ADMIN Capability Error

**Challenge**: The pod failed with a “missing NET_ADMIN” error.  
**Solution**: I added securityContext: capabilities: add: ["NET_ADMIN"] to the YAML and redeployed with microk8s kubectl apply -f pihole-deployment.yaml.

### 4. Docker Permission Issues

**Challenge**: docker ps -a required sudo.  
**Solution**: I applied group changes with sudo usermod -aG docker $USER && newgrp docker.

## Performance Verification

I confirmed functionality through tests: I ran nslookup doubleclick.net 192.168.1.200, which returned 0.0.0.0 to confirm blocking, nslookup x.com 192.168.1.200, which resolved to 172.66.0.227, and a resilience test with microk8s kubectl delete pod -l app=pihole followed by microk8s kubectl get pods, showing a new pod running within 30 seconds. I also verified DHCP leases at http://192.168.1.200/admin.

## Key Achievements

- Deployed Pi-hole in Kubernetes on Raspberry Pi 5  
- Replaced router DNS/DHCP with a containerized solution  
- Achieved pod auto-restart with Kubernetes Deployment  
- Resolved complex networking and container issues  
- Built a cloud-ready project for resume showcase

## Skills Demonstrated

Through this project, I demonstrated skills in:  
- Cloud & container management (Docker, Kubernetes)  
- Network configuration (static IP, DNS, DHCP, Netplan)  
- Linux system administration (Ubuntu, service management)  
- Troubleshooting methodology (diagnosing DNS/DHCP issues)  
- Technical documentation

## Future Improvements

Potential enhancements include:  
- Adding a Kubernetes Service for external access  
- Implementing persistent volume claims (PVCs)  
- Automating deployment with Helm charts  
- Testing full Pi reboot recovery

## Resources

- [Official Pi-hole Documentation](https://docs.pi-hole.net/)  
- [MicroK8s Documentation](https://microk8s.io/docs)  
- [Ubuntu Netplan Documentation](https://netplan.readthedocs.io/en/latest/)  
- [Raspberry Pi Documentation](https://www.raspberrypi.com/documentation/)
