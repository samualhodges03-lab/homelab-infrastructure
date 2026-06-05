# Enterprise Environment Lab Documentation

* **Domain Scope:** `lab.lan`

* **Hypervisor:** Proxmox VE

* **Author:** Samual Hodges

* **Last Updated:** June 4, 2026

## 1. Network Segmentation & Firewall Topology

### 1.1 VLAN Matrix

| VLAN ID | Segment Name | Subnet Range    | Gateway      | Function / Assets                |
| ------- | ------------ | --------------- | ------------ | -------------------------------- |
| **10**  | `MGMT`       | `10.10.10.0/24` | `10.10.10.1` | Jump Box, Hypervisor UI          |
| **20**  | `CORE`       | `10.20.20.0/24` | `10.20.20.1` | Windows Server DC (`CORE-DC01`)  |
| **30**  | `PROD`       | `10.30.30.0/24` | `10.30.30.1` | Ubuntu Server (`PROD-UBUNTU-01`) |

### 1.2 OPNsense Firewall Rules

#### MGMT

* `Pass` `in` `IPv4` `any` `LAN network` `any`
  
  * Default allow LAN to any rule.
- `Pass` `in` `IPv6` `any` `Lan network` `any`
  
  - Default allow LAN IPv6 to any rule.

#### CORE

* `Pass` `in` `IPv4` `any` `CORE_VLAN network` `PROD_VLAN network`
  
  * Allows CORE network to manage PROD.
- `Pass` `in` `IPv4` `any` `CORE_VLAN network` `CORE_VLAN address`
  
  - Allows CORE access to Local OPNsense Gateway Interface.

- `Pass` `in` `IPv4` `any` `CORE_VLAN network` `RFC1918_Networks` `Invert Destination`
  
  - Allows CORE network internet access while blocking private networks.

#### PROD

* `Pass` `in` `IPv4` `any` `PROD_VLAN network` `CORE_VLAN network`
  
  * Allows PROD network access to CORE network.
- `Pass` `in` `IPv4` `any` `PROD_VLAN network` `RFC1918_Network` `Invert Destination`
  
  - Allows PROD access to internet while blocking private networks.

## 2. Centralized Identity Governance & RBAC

### 2.1 Active Directory & Core Services Topology

* **Root Domain:** `lab.lan`

* **Primary Domain Controller:** `CORE-DC01`

* **Operating System:** Windows Server 2025 Standard (Evaluation)

* **Logical Network Placement:** CORE Zone (VLAN 20)

* **Static Address:** `10.20.20.10/24`

* **Default Gateway:** `10.20.20.1` (OPNsense Subinterface)

#### DNS

* **Primary DNS:** `127.0.0.1`
  
  * AD relies on local SRV records to locate domain services.

* **Upstream Forwarder:** `10.20.20.1`
  
  * Handles external web resolution.

* **Reverse Lookup Zone:** `20.20.10.in-addr.arpa`
  
  * Facilitates reverse lookups via PTR records.

### 2.2 Identity Access Management

#### Organizational Unit Taxonomy

* **lab.lan** (Root Domain)
  
  * **Lab_Infrastructure** (OU)
    
    * **Identity_Accounts** (OU)
      
      * `Linux_Admins` (Security Group)
      
      * `samual`(User Account)
      
      * `lab_worker` (User Account)
    - **Linux_Assets** (OU)
      
      - `PROD-UBUNTU-01` (Computer Object)

### 2.3 Role-Based Access Control (RBAC) Translation

* **Role:** Linux Systems Administrator

* **AD Group:** Linux_Admins

* **Target Authorization:** Full Passwords/Sudo Root Privileges.

* **Assigned Context:** samual

## 3. Computer Node Infrastructure (PROD-UBUNTU-01)

* **Operating System:** Ubuntu Server 26.04 LTS

* **Static IP:** 10.30.30.10

* **DNS Resolver:** 10.20.20.10

* **Search Domain:** lab.lan

### 3.1 Cross-Platform Integration Stack

Maps Actice Directory accounts to the Linux server using System Security Services Daemon (sssd) and realmd.

`/etc/sssd/sssd.conf`

```toml
[sssd]
domains = lab.lan
config_file_version = 2
services = nss, pam

[domain/lab.lan]
default_shell = /bin/bash
krb5_store_password_if_offline = True
cache_credentials = True
krb5_realm = LAB.LAN
realmd_tags = manages-system joined-with-adcli
id_provider = ad
fallback_homedir = /home/%u@%d
ad_domain = lab.lan
use_fully_qualified_names = False
ldap_id_mapping = True
access_provider = ad
```

### 3.2 Linux Security Policy Enforcement

Missing user directories built on the fly during initial domain authentications.

```bash
sudo pam-auth-update --enable mkhomedir
```

Root permissions mapped directly to Actice Directory groups using visudo.

```markup
%Linux_Admins@lab.lan ALL=(ALL:ALL) ALL
```

## 4. Containerization & Storage Architecture

### 4.1 Storage Matrix

* `/opt/docker-containers/`(Root Directory | Owner: root:docker | Permissions: 775)
  
  * `infra-stack/`
    
    * `docker-compose.yml`
    
    * `portainer-data/`(Persistent Database Volume for Control Plane)
    
    * `proxy-data/`(Nginx Proxy Manager Static Rules)
    
    * `proxy-letsencrypt/`(SSL Certificate Cryptograpic Storage)

### 4.2 Application Authorization Delegation

The `samual`user is given permission to manipulate Docker runtimes without invoking total root privileges.

```bash
sudo gpasswd -a samual docker
```

## 5. Reverse Proxy & Application Orchestration

### 5.1 Infrastructure Stack

```yaml
version: '3.8'

services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: infra-portainer
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /opt/docker-containers/infra-stack/portainer-data:/data
    networks:
      - default
      - proxy-tier

  nginx-proxy-manager:
    image: 'jc21/nginx-proxy-manager:latest'
    container_name: infra-proxy
    restart: unless-stopped
    ports:
      - '80:80'    # Incoming HTTP Edge Route
      - '443:443'  # Incoming HTTPS Edge Route
      - '81:81'    # Administration UI Endpoint
    volumes:
      - ./proxy-data:/data
      - ./proxy-letsencrypt:/etc/letsencrypt
    networks:
      - default
      - proxy-tier

networks:
  proxy-tier:
    external: true
```

### 5.2 Edge Routing Map & DNS Records

Applications are exposed via Authoritative Active Directory DNS Forward Lookup Zone A-records point to Nginx Proxy Manager.

| Target Service | Local Domain Host Name | Internal Target Network Destination | Listening Port |
| -------------- | ---------------------- | ----------------------------------- | -------------- |
| Portainer CE   | `portainer.lab.lan`    | `https://infra-portainer`           | `9443`         |
| Proxy UI       | Direct IP              | `http://10.30.30.10`                | `81`           |
