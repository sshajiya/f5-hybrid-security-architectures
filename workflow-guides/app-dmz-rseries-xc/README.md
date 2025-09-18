# Extending App DMZ to Global Service Tier with F5 rSeries and Distributed Cloud Services

# Table of Contents

- [Extending App DMZ to Global Service Tier with F5 rSeries and Distributed Cloud Services](#extending-app-dmz-to-global-service-tier-with-f5-rseries-and-distributed-cloud-services)
- [Table of Contents](#table-of-contents)
- [Overview](#overview)
- [Setup](#setup)
- [1. Initial preparations](#1-initial-preparations)
  - [1.1 Requirements](#11-requirements)
  - [1.2 Configure Application VMs](#12-configure-application-vms)
  - [1.3 Deploy and Configure BIG-IP on F5 rSeries](#13-deploy-and-configure-big-ip-on-f5-rseries)
    - [1.3.1 Deploy BIG-IP on F5 rSeries](#131-deploy-big-ip-on-f5-rseries)
    - [1.3.2 Configure BIG-IP on F5 rSeries](#132-configure-big-ip-on-f5-rseries)
    - [1.3.3 Create BIG-IP Virtual Server](#133-create-big-ip-virtual-server)
- [2. Configure Environment](#2-configure-environment)
  - [2.1 Deploy CE Tenant on VMware](#21-deploy-ce-tenant-on-vmware)
    - [2.1.1 Create Secure Mesh Site in F5 Distributed Cloud](#211-create-secure-mesh-site-in-f5-distributed-cloud)
    - [2.1.2 Configure the Second Cluster](#212-configure-the-second-cluster)
  - [2.2 Configure XC Virtual Site](#22-configure-xc-virtual-site)
- [3. Expose Application to the Internet](#3-expose-application-to-the-internet)
  - [3.1 Create the HTTP Load Balancer](#31-create-the-http-load-balancer)
- [4. Protect Application](#4-protect-application)
  - [4.1 Configure WAF](#41-configure-waf)
  - [4.2 Configure Bot Protection](#42-configure-bot-protection)
  - [4.3 Configure API Discovery](#43-configure-api-discovery)
  - [4.4 Configure DDoS Protection](#44-configure-ddos-protection)
  - [4.5 Configure Malicious User and IP Reputation](#45-configure-malicious-user-and-ip-reputation)
  - [4.6 Verify Application](#46-verify-application)
- [5. Setup DMZ Configuration](#5-setup-dmz-configuration)
- [6. Conclusion](#6-conclusion)

# Overview

This guide provides steps for a comprehensive Demilitarized Zone (DMZ) setup using the F5 Distributed Cloud Services (XC) environment, the F5 Hardware (F5 rSeries), and VMware.

**Challenges**

A DMZ is a physical or logical subnetwork that contains and exposes an organization's external-facing services to an untrusted network, typically the Internet. It serves as a buffer zone between the secure internal network and external networks, providing an additional layer of security to ensure that potentially malicious traffic does not reach critical internal systems directly.

To make data center applications accessible on the internet, IT teams traditionally handle several networking operations, including:

- **NAT** (Network Address Translation): Converting public IP addresses to private server IPs.
- **DNS Resolution**: Ensuring domain names resolve to the correct IP addresses.
- **Load Balancing**: Distributes incoming application traffic across multiple servers to ensure reliability, optimal resource utilization, and high availability.
- **Security Operations**: Deploying protections such as Web Application Firewalls (WAF) and Distributed Denial of Service (DDoS) mitigation.

![rseries](./assets/rseries-before-v2.png)

Handling these operations at the App Services tier and on a per-application basis adds complexity, making application delivery more challenging, such as:

- Management and "stitching" of multiple app DMZ environments at scale
- Standardized, consistent policies across data center deployments
- Handling unwanted traffic (bad actors and bots) at the global services tier

These challenges compound into serious problems for modern microservices application architectures - particularly **handling unwanted/unfiltered traffic** to app services and API endpoints, and **having no uniform security or visibility** when managing multiple sites (multiple data centers and/or clouds).

**Solution**

Distributed Cloud Services simplify these challenges by providing centralized security services, which include volumetric DDoS protection, API protection, and bot mitigation as part of WAF configurations at the network edge. This is enabled by installing and operating a Customer Edge (CE) in various on‑premises environments, such as VMware, OpenShift, Nutanix.

(1) **Global Services Tier**

- Keeps unwanted traffic off your infrastructure
- Broad-spectrum volumetric DDoS mitigation (L3/L4)
- Anti-abuse including bot/fraud detection and mitigation
- Ease of securely exposing applications to the public internet by simplifying or eliminating manual handling of routing and other networking tasks
- Simplifying workflows for pushing out App DMZ toward the network edge
- Standard company app security policy / policies used by all apps

(2) **App Services Tier**

- Retains important, integrated security controls and policies, including automation and CI/CD pipelines, at your app
- Workload-specific security policy definitions & enforcement
- Closest to the application & Line of Business / security teams managing specific app services

![rseris](./assets/diagram-overview-2.png)

# Setup

The objective of this setup is to create a secure DMZ environment for the application using the F5 hardware platform, which provides a modern topology for flexible and scalable network connectivity, enhanced performance, and protection, together with VMware. The diagram below shows high-level components and their interactions. The setup includes two data centers, each with an origin pool that connects to the F5 Distributed Cloud site deployed in VMware. The Distributed Cloud site is connected to the BIG-IP, where a Virtual Server is configured. Our sample app (Arcadia) is inside the BIG-IP Virtual Server pool. The application is protected by a Web Application Firewall (WAF), DDoS Protection, Bot Protection, and API Discovery.

The setup flow includes the following:

- Configuration of BIG-IP on F5 rSeries
- Configuration of data centers with Customer Edge (CE) Sites on F5 rSeries and VMware
- Demo application deployment in the data center
- F5 Distributed Cloud Secure Mesh Site configuration and combining them into a single Virtual Site
- Application exposure to the Internet using an HTTP Load Balancer
- Application protection with a Web Application Firewall (WAF), DDoS Protection, Bot Protection, and API Discovery
- Upgrading the solution with a second data center and configuring an HTTP Load Balancer for a complete DMZ configuration

# 1. Initial preparations

## 1.1 Requirements

The following components are required to complete the setup:

- Access to the [Distributed Cloud Services](https://cloud.f5.com) with the following services enabled:
  - Ability to create Sites
  - Bot Protect
  - DDoS Protect
  - API Protect
  - API Discovery
- F5 rSeries: 5600 / 5800 / 5900/ 10600 / 10800 / 10900 / 12600 / 12800 / 12900
- Ubuntu VM with access to the F5 rSeries network
- VMware vCenter
- Domain name

The following diagram shows the components and network configuration of the setup:

![rseries](./assets/diagram-configure.png)

## 1.2 Configure Application VMs

The main application is a simple web application that simulates a banking application. The application is hosted on an Ubuntu VM. The following steps are required to configure the main application VM:

- SSH into the VM
- Install [Docker and Docker Compose](https://docs.docker.com/engine/install/ubuntu/)
- Clone the repository
- Open `./application/main/` folder
- Run `docker compose up -d`

Optionally update the environment variables in the `docker-compose.yml` file.

Verify that the application is running by accessing `http://{{your_vm_ip}}:8080` in the browser or by using a curl command.

![Secure Mesh Site](./assets/vmware_app.png)

## 1.3 Deploy and Configure BIG-IP on F5 rSeries

### 1.3.1 Deploy BIG-IP on F5 rSeries

Download the BIG-IP image for F5OS from the [F5 Downloads](https://my.f5.com/manage/s/downloads) site and save it to your local machine.

In the F5 rSeries interface navigate to `Tenant Images` and click on the `Upload` button.

![rseries-bigip](./assets/f5os_bigip_tenant_image.png)

Select the BIG-IP image and click `Open`.

![rseries-bigip](./assets/f5os_bigip_tenant_image_upload.png)

Navigate to `Tenant Deployments` and click on the `Add` button.

![rseries-bigip](./assets/f5os_bigip_create.png)

Fill in the required fields:

- `Name`: big-ip-tmos
- `Type`: BIG-IP
- `Image`: select the image you uploaded
- `IP Address`: IP address of the BIG-IP management interface
- `Gateway`: Gateway IP address
- `VLANs`: check the `XC-SLI` and `BIG-IP` VLANs
- `vCPUs`: 4
- `Virtual Disk Size`: 82 GB
- `State`: Deployed

Click on the `Save & Close` button to apply the changes.

![rseries-bigip](./assets/f5os_bigip_create_part_1.png)

![rseries-bigip](./assets/f5os_bigip_create_part_2.png)

### 1.3.2 Configure BIG-IP on F5 rSeries

Log in to your BIG-IP TMOS instance and navigate to `Network`. Select `VLANs` and click the `Create` button.

![rseries-bigip](./assets/bigip_config_vlan_navigate.png)

In the form, fill in the VLAN name, tag, and interface. Click `Finished` when the fields are filled out.

![rseries-bigip](./assets/bigip_config_vlan_create.png)

Next, proceed to `Self IPs` and click `Create`.

![rseries-bigip](./assets/bigip_config_selfip_navigate.png)

This opens the configuration form. Fill in the following fields:

- `Name`: 10.5.11.20
- `IP Address`: 10.5.11.20 (or any other IP address you want to assign in F5 Distributed Cloud SLI network)
- `Netmask`: 255.255.255.0
- `VLAN / Tunnel`: select the VLAN you created in the previous step

Click `Finished` as soon as the fields are filled out.

![rseries-bigip](./assets/bigip_config_selfip_create.png)

### 1.3.3 Create BIG-IP Virtual Server

In this section, we will configure the BIG-IP Virtual Server to expose the application to the Distributed Cloud SLI network. We will create a pool with the application VM as a member and then create a Virtual Server to route the traffic to the pool.

Open the BIG-IP interface and navigate to the `Local Traffic` tab. Click on the `Pools` and then click on the `Create` button.

![bigip](./assets/bigip_pool_create.png)

Fill in the required fields:

- `Name`: application-pool
- `Health Monitors`: select `http`
- `Node Name`: give a name to the node
- `Address`: IP address of the application VM
- `Service Port`: 8080

Click on the `Add` button to add the node to the pool and then click on the `Finished` button to create the pool.

![bigip](./assets/bigip_pool_details.png)

In the `Local Traffic` tab click on the `Virtual Servers`. Then click on the `Create` button.

![bigip](./assets/bigip_vs_navigate.png)

Fill in the required fields:

- `Name`: arcadia-application
- `Destination Address`: select the `SLI` network IP address
- `Service Port`: 8080
- Set `HTTP Profile (Client)` to `http`

![bigip](./assets/bigip_vs_name.png)

- Set `Source Address Translation` to `Auto Map`

![bigip](./assets/bigip_vs_map.png)

- Set `Default Pool` to `application-pool` and click on the `Finished` button to create the Virtual Server.

![bigip](./assets/bigip_vs_pool.png)

The application is now exposed to the Distributed Cloud SLI network. You can try to access the application using the IP address of the SLI network.

# 2. Configure Environment

## 2.1 Deploy CE Tenant on VMware

In this section, we will create a Secure Mesh Site v2 in F5 Distributed Cloud and deploy it in a VMware data center. The Secure Mesh Site will connect to the BIG-IP Virtual Server we created in the previous section.

### 2.1.1 Create Secure Mesh Site in F5 Distributed Cloud

Open the F5 Distributed Cloud console and navigate to `Multi-Cloud Network Connect`. In the left navigation pane, click `Site Management`, then click `Secure Mesh Sites v2`. On the `Secure Mesh Sites v2` page, click `Add Secure Mesh Site`.

![vmware-01](./assets/vmware-01.png)

Enter a site name and assign the custom label `dc == dc1-dmz`.

![vmware-02](./assets/vmware-02.png)

Select the provider as `VMware`. Leave other fields at their defaults.

![vmware-03](./assets/vmware-03.png)

Click `Add Secure Mesh Site` to apply the changes.

![vmware-04](./assets/vmware-04.png)

Open the action menu of the created site and click `Download Image`. Save the image to your local machine. You will need it later.

![vmware-05](./assets/vmware-07.png)

While the image is downloading, open the action menu again and click `Generate Node Token`.

![vmware-06](./assets/vmware-06.png)

Click `Copy Token` in the `Generate Node Token` dialog to save the token to your clipboard.

![vmware-07](./assets/vmware-05.png)

Open VMware vCenter or ESXi (in our case) and start the deployment of the downloaded image. Create a new VM and select `Deploy a virtual machine from an OVF or OVA file`. Click `Next` to proceed.

![vmware-08](./assets/vmware-08.png)

Enter a name for the VM and select the downloaded image. Then click `Next` to proceed.

![vmware-09](./assets/vmware-09.png)

Select the storage and click `Next`.

![vmware-10](./assets/vmware-10.png)

Select the Outside network and set disk provisioning to `Thin`. Optionally, select `Power on automatically`, then click `Next`.

![vmware-11](./assets/vmware-11.png)

Enter the node name, the token you copied earlier, an admin password, and network settings. Click `Next` to proceed.

![vmware-12](./assets/vmware-12.png)

Review the configuration and click `Finish` to deploy the VM.

![vmware-13](./assets/vmware-13.png)

Open the VM settings and add a new network adapter. Select the `Inside Network` and click `Save` to apply the changes.

![vmware-14](./assets/vmware-14.png)

After the VM is deployed and powered on, access the console to verify that the site is connected to the XC Cloud. It may take a few minutes for the site to connect. The site status will change to `Online` once the site is ready.

![vmware-15](./assets/vmware-15.png)

Now, open the action menu of the created site and click `Manage Configuration` to open the site details.

![vmware-16](./assets/vmware-16.png)

Click the `Edit Configuration` button to enable editing mode.

![vmware-17](./assets/vmware-17.png)

Find `node-0` in the list and click the `Edit` button.

![vmware-18](./assets/vmware-18.png)

In the `Interfaces` section, click `Add Item` to add a new interface.

![vmware-19](./assets/vmware-19.png)

Enter the interface name.

![vmware-20](./assets/vmware-20.png)

Then select `VLAN Interface` as the interface type, set the parent interface to `esp224`, and the VLAN to `511`. Set the static IP address.

NOTE: Network settings may differ in your environment.

![vmware-21](./assets/vmware-21.png)

Select `Site Local Inside (Local VRF)` as the VRF and click `Apply` to apply the changes.

![vmware-22](./assets/vmware-22.png)

Verify that the interface is added and click `Apply` to apply the changes.

![vmware-23](./assets/vmware-23.png)

Click `Save Secure Mesh Site` to save the configuration.

![vmware-24](./assets/vmware-24.png)


### 2.1.2 Configure the Second Cluster

Repeat the steps from the [1.3 Deploy and Configure BIG-IP on F5 rSeries](#13-deploy-and-configure-big-ip-on-f5-rseries) section and from the [2.1 Deploy CE Tenant on VMware](#21-deploy-ce-tenant-on-vmware) section to configure the second rSeries device.

## 2.2 Configure XC Virtual Site

To simplify application management, we will create a Virtual Site in the XC Cloud and assign the Secure Mesh Site to it. This allows us to access the application using the Virtual Site name and scale by adding more Secure Mesh Sites in the future.

![rseries](./assets/diagram-vsite.png)

Let's start by adding a virtual site. In the F5 Distributed Cloud console, navigate to the **Shared Configuration** service. From there, select **Virtual Sites** and click **Add Virtual Site**.

![Virtual Site](./assets/virtual_site_add.png)

In the form, give the virtual site the same name you specified as the [label](#211-create-secure-mesh-site-in-f5-distributed-cloud) for Secure Mesh Sites. Select the **CE** site type. Then add a selector expression specifying the label value, and click **Save and Exit**.

![Virtual Site](./assets/virtual_site_config.png)

# 3. Expose Application to the Internet

![rseries](./assets/diagram-before.png)

## 3.1 Create the HTTP Load Balancer

Next, we will configure the HTTP Load Balancer to expose the created Virtual Site to the Internet.

![rseries](./assets/diagram-httplb.png)

Proceed to the **Multi-Cloud App Connect** service => **Load Balancers** => **HTTP Load Balancers**. Click **Add HTTP Load Balancer**.

![HTTP LB](./assets/http_lb_create.png)

First, give the HTTP Load Balancer a name.

![HTTP LB](./assets/http_lb_name.png)

Then configure the **Domains and LB Type** section. Enter the **arcadia-dmz.f5-cloud-demo.com** domain and select **HTTPS with Automatic Certificate** as the Load Balancer Type. Make sure to enable HTTP redirect to HTTPS and add the HSTS header.

![HTTP LB](./assets/http_lb_domain.png)

Scroll down to the **Origins** section and add an origin pool by clicking the **Add Item** button.

![HTTP LB](./assets/http_lb_origin.png)

Open the **Origin Pool** drop-down menu and click **Add Item** to add an origin pool.

![HTTP LB](./assets/http_lb_add_pool.png)

Give the origin pool a name.

![HTTP LB](./assets/http_lb_pool_name.png)

Then click **Add Item** to add an origin server.

![HTTP LB](./assets/http_lb_pool_origin.png)

Select **IP address of Origin Server on given Sites** as the Origin Server type and enter the **10.5.11.20** private IP (your BIG-IP XC interface). Then, in the drop-down menu, select the [Virtual Site](#22-configure-xc-virtual-site) we created earlier. Complete the configuration by clicking **Apply**.

![HTTP LB](./assets/http_lb_pool_details.png)

Configure the second origin server for the second BIG-IP instance in the same way. The IP address is **10.5.11.21**.

Type in the **8080** origin server port.

![HTTP LB](./assets/http_lb_pool_port.png)

Scroll down to the **Health Checks** section and click the **Add Item** button to add a health check.

![HTTP LB](./assets/http_lb_health_add.png)

Give the health check a name and leave the default settings. Then click **Continue** to save the health check configuration.

![HTTP LB](./assets/http_lb_health_details.png)

Scroll down and click **Continue**.

![HTTP LB](./assets/http_lb_pool_save.png)

**Apply** origin pool configuration.

![HTTP LB](./assets/http_lb_pool_apply.png)

Now that the HTTP Load Balancer is configured, click **Save and Exit** to save it.

![HTTP LB](./assets/http_lb_save.png)

# 4. Protect Application

Now that we have exposed the Virtual Site to the Internet using an HTTP Load Balancer, we will configure protection for the deployed application: WAF, Bot Protect, API Discovery, DDoS Protection, and Malicious User and IP Reputation.

![rseries](./assets/diagram-waf.png)

To do that, go back to the F5 Distributed Cloud console and select **Manage Configuration** in the service menu of the created HTTP Load Balancer.

![Configure](./assets/configure_manage.png)

Click the **Edit Configuration** button to enable the editing mode.

![Configure](./assets/configure_edit.png)

## 4.1 Configure WAF

First, let's configure WAF protection. Scroll down to the **Web Application Firewall** section and enable WAF. Open the drop-down menu and click **Add Item**.

![Configure](./assets/configure_waf.png)

Give WAF a name and move on to **Enforcement Mode** configuration.

![Configure](./assets/configure_waf_name.png)

Select **Blocking** mode in the drop-down menu to log and block threats.

![Configure](./assets/configure_waf_mode.png)

Proceed to **Detection Settings**. Select a **Custom** Security Policy and review its settings. Then scroll down to **Signature-Based Bot Protection** and select **Custom**.

![Configure](./assets/configure_waf_detection.png)

Finally, configure the **Blocking Response Page** in **Advanced configuration**. Select **Custom** and configure as needed. Click **Continue** to complete WAF configuration and return to the HTTP configuration page.

![Configure](./assets/configure_waf_advanced.png)

## 4.2 Configure Bot Protection

Next, configure Bot Protection. Scroll to the **Bot Protection** section and select **Enable Bot Defense Standard** in the drop-down menu. Click **Configure** to continue.

![Configure](./assets/configure_bot.png)

Proceed to configure Protected App Endpoints.

![Configure](./assets/configure_bot_app_endpoints.png)

Click the **Add Item** button which will open the creation form.

![Configure](./assets/configure_bot_add_endpoint.png)

Configure the endpoint: give it a name, select the HTTP methods, and specify the endpoint label category. Specify **Authentication** as the flow label category and select **Login** for the flow label. Specify the path prefix **/trading/auth**. Select **Block** for the Bot Mitigation action and save the configuration by clicking **Apply**.

![Configure](./assets/configure_bot_endpoint_config.png)

Take a look at the created App Endpoint and apply its configuration.

![Configure](./assets/configure_bot_endpoint_apply.png)

You will see the Bot Defense Policy settings. Click **Apply** to proceed.

![Configure](./assets/configure_bot_apply.png)

Now that the Bot Protection is configured for the HTTP Load Balancer, we can move on to API Discovery.

## 4.3 Configure API Discovery

In the **API Protection** section, enable API Discovery and enable learning from redirected traffic. Once the configuration is ready, proceed to the DDoS settings.

![Configure](./assets/configure_api.png)

## 4.4 Configure DDoS Protection

Go to the **DoS Protection** section and select serving a JavaScript challenge to suspicious sources. Then select **Custom** for Slow DDoS Mitigation.

![Configure](./assets/configure_ddos.png)

## 4.5 Configure Malicious User and IP Reputation

In the **Common Security Controls** section, enable the IP Reputation service and Malicious User Detection. Then select **JavaScript Challenge** for this HTTP LB.

![Configure](./assets/configure_other.png)

The security configuration is complete. Review it and click **Save and Exit**.

![Configure](./assets/configure_save.png)

## 4.6 Verify Application

Now that all protections are configured, we can verify the application. Access the application using the domain name specified in the [Create HTTP Load Balancer](#31-create-the-http-load-balancer) section.

To verify WAF protection, access the application using a browser or a curl command and check if the request is blocked by WAF. Let's simulate a simple XSS attack by adding a script tag to the request. Open the browser and navigate to the application URL `https://arcadia-dmz.f5-cloud-demo.com?param=<script>alert('XSS')</script>`. You should see the WAF blocking page.

![Configure](./assets/test_xss.png)

To verify Bot Protection, access the application using a browser or a curl command and check if the request is blocked by Bot Protection. Let's simulate a bot attack by sending a request to the protected endpoint. Open the terminal and run the following command `curl -i -X POST https://arcadia-dmz.f5-cloud-demo.com/trading/auth`. You should see the Bot Protection blocking page in the response.

```bash
curl -i -X POST https://arcadia-dmz.f5-cloud-demo.com/trading/auth

HTTP/2 403
server: volt-adc
strict-transport-security: max-age=31536000
cache-control: no-cache
content-type: text/html; charset=UTF-8
pragma: no-cache
x-volterra-location: pa4-par
content-length: 71
date: Wed, 31 Jul 2024 23:00:45 GMT

The requested URL was rejected. Please consult with your administrator.
```

To verify API Discovery, open the `docker-compose.yml` file and replace the `BASE_URL` variable with the application URL. Then run `docker compose up -d` from the `docker/api_discovery` directory.
This will send a request to the application and trigger the API Discovery. It will take some time for the API Discovery to learn the traffic. After that, you can check the API Discovery dashboard in the F5 Distributed Cloud Console.

```bash
version: '3'
services:
  openbanking-traffic:
    image: public.ecr.aws/r2r2l6v3/arcadia-finance/openbanking-traffic:v0.0.2
    environment:
      BASE_URL: https://{{your-domain-here}}/openbanking
```

Navigate to the **Applications** tab and select your HTTP Load Balancer. Then click the **API Endpoints** tab to see the learned API endpoints. Change the view to **Graph** to see the API endpoints graph.

![Configure](./assets/http_discovery.png)

# 5. Setup DMZ Configuration

Finally, we will configure the HTTP Load Balancer by creating the second origin pool for the second data center and configuring it.

![rseris](./assets/diagram-dmz.png)

This setup requires a second data center with the same configuration as the first one. Repeat the steps from [1. Initial preparations](#1-initial-preparations) and [2. Configure Environment](#2-configure-environment) to create a second data center with the same components. Then create a Virtual Site for the second data center as described in [2.2 Configure XC Virtual Site](#22-configure-xc-virtual-site).

Once the second data center is ready, proceed with the configuration. Go to the F5 Distributed Cloud console and select **Manage Configuration** in the service menu of the earlier [created HTTP Load Balancer](#31-create-the-http-load-balancer).

![Second DC](./assets/dc2_manage.png)

Click the **Edit Configuration** button to enable the editing mode.

![Second DC](./assets/dc2_edit.png)

Scroll to the **Origins** section and click **Add Item**. This opens the origin pool creation form.

![Second DC](./assets/dc2_add_pool.png)

Open the **Origin Pool** drop-down menu and click **Add Item** to add an origin pool.

![Second DC](./assets/dc2_create_pool.png)

Give the origin pool a name.

![Second DC](./assets/dc2_pool_name.png)

Then click **Add Item** to add an origin server.

![Second DC](./assets/dc2_add_origin.png)

Select **IP address of Origin Server on given Sites** as the Origin Server type and enter the **10.6.11.20** private IP (your BIG-IP XC interface in the second data center). Then, in the drop-down menu, select the second Virtual Site. Complete the configuration by clicking **Apply**.

![Second DC](./assets/dc2_configure_origin.png)

Create a second origin server for the second BIG-IP instance. In our case, the IP address is **10.6.11.21**.

Type in the **8080** origin server port.

![Second DC](./assets/dc2_port.png)

Scroll down and click **Continue**.

![Second DC](./assets/dc2_save_pool.png)

Change the **Priority** of the origin pool to **0**. This makes the second origin pool a backup for the first one. **Apply** the origin pool configuration.

![Second DC](./assets/dc2_apply_pool.png)

The second configured origin pool will appear on the list.

![Second DC](./assets/dc2_pool_result.png)

Now that we have added and configured the second origin pool of the HTTP Load Balancer for the second data center, click **Save and Exit** to save it.

# 6. Conclusion

In this guide, we configured a comprehensive DMZ setup using F5 rSeries hardware, BIG-IP, and the F5 XC Cloud environment. We deployed a simple web application, configured BIG-IP and the XC CE Tenant on F5 rSeries, exposed the application to the Internet using an HTTP Load Balancer, and protected the application with WAF, Bot Protection, API Discovery, DDoS Protection, and Malicious User and IP Reputation. We also configured the second data center and added it to the HTTP Load Balancer as a backup origin pool.

This setup provides a secure and scalable environment for exposing on‑prem applications to public Internet access, with advanced protection and networking management performed at the network edge using F5 Distributed Cloud Services. The F5 rSeries hardware and F5 XC Cloud environment provide a powerful platform for deploying and managing application networking and security in a scalable and efficient way.
