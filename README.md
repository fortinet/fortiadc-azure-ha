# FortiADC HA

This project contains the code and templates for the **Microsoft Azure** FortiADC HA deployments.

## Azure Portal

<a href="https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Ffortinet%2Ffortiadc-azure-ha%2Fmain%2Ftemplates%2Fdeploy_fadc_ha.json"
target="_blank">
<img src="https://raw.githubusercontent.com/Azure/azure-quickstart-templates/master/1-CONTRIBUTION-GUIDE/images/deploytoazure.svg?sanitize=true"/>
</a><a href="http://armviz.io/#/?load=https%3A%2F%2Fraw.githubusercontent.com%2Ffortinet%2Ffortiadc-azure-ha%2Fmain%2Ftemplates%2Fdeploy_fadc_ha.json" target="_blank">
<img src="https://raw.githubusercontent.com/Azure/azure-quickstart-templates/master/1-CONTRIBUTION-GUIDE/images/visualizebutton.svg?sanitize=true"/>
</a>


## Overview of FortiADC HA pairing on the Azure platform

Microsoft Azure supports FortiADC HA (High Availability) in Active-Active-VRRP mode. Azure can only support
the VRRP HA mode because other modes that require access to Layer 2 are inaccessible in Public Cloud
environments. For more information on FortiADC HA modes or the Azure Virtual Network, see the [FortiADC
Handbook on High Availability Deployments](https://docs.fortinet.com/document/fortiadc/latest/handbook/856481/chapter-15-high-availability-deployments) and the [Azure Virtual Network FAQ](https://docs.microsoft.com/en-gb/azure/virtual-network/virtual-networks-faq).

There are 2 methods to deploy the FortiADC VRRP HA on Azure:
1. Deploy the VRRP HA using an API call to Azure
2. Deploy the VRRP HA using Azure Load Balancers

Introduced in the FortiADC 6.2.0 release, the method of using Azure Load Balancers to pair FortiADC HA on the Azure platform was developed to resolve the issues created by the Azure API call. 

Previously, VRRP HA on Azure used the API call to Azure to migrate the public IPs associated with the FortiADC-VM on the Azure backend in the event of HA failover. The IP migration process may take several minutes during the HA failover, causing business traffic to break until the migration is complete. With using Azure Load Balancers, the downtime for the FortiADC member may only be seconds during the HA failover because it does not require IP migration.


## Before You Begin

You need to meet the following prerequisites to deploy HA on Azure:
- A Microsoft Azure account that can be used to log into the Azure portal. If you do not have an Azure
account, please follow the instructions on the Microsoft Azure website on how to obtain one.
- A Microsoft Azure subscription that allows you to purchase the FortiADC-VM and launch in a desired
location.
- Valid FortiADC licenses if BYOL image type is chosen (at least a minimum of two CPUs) for all HA members.

## VRRP HA using API call to Azure

In this VRRP HA deployment mode that uses an API call to Azure, each FortiADC network interface on Azure maintains its own IP configuration table. You can add secondary IPs in the IP configuration table and use the
IPs in the FortiADC configuration, such as virtual server IPs.

:warning: Prior to the FortiADC 6.2.0 release, deploying VRRP HA on Azure using an API call was the standard deployment mode. In version 6.2.0, deploying VRRP HA using Azure Load Balancers was introduced to mitigate the issues caused by the Azure API call deployment mode. We recommend deploying VRRP HA on Azure using Azure Load Balancers as the preferred deployment mode.

In the Figure 1 example below, there are two (2) FortiADC-VMs in HA VRRP mode that share the same configuration for Virtual Server, Virtual Server NAT Pool, Firewall NAT SNAT, Firewall 1-1 NAT, and the network interface floating IP. On the Azure platform side, the corresponding IP configuration is only attached to the working FprtiADC, which is the FAD-HA-VM1 in this example.

![](https://github.com/fortinet/fortiadc-azure-ha/blob/main/figures/Figure%201%20FortiADC%20HA%20pair%20on%20Azure%20platform.png?raw=true)

> Figure 1 FortiADC HA pair on Azure platform.

In the event of HA failover, the FAD-HA-VM2 becomes the working node. This requires the IP configuration to be migrated from FAD-HA-VM1 to FAD-HA-VM2 in the Azure backend. More specifically, this disassociates the related IP configuration on the FAD1-OutSideNic / FAD1-InSideNic and associates them on the FAD2-OutSideNic / FAD2-OutSideNic. This IP migration process is executed using an Azure API call, as illustrated in the Figure 2 example below.

![](https://github.com/fortinet/fortiadc-azure-ha/blob/main/figures/Figure%202%20IP%20Migration%20between%20the%20Azure%20network%20interface%20when%20HA%20failover%20occurs.png?raw=true)

> Figure 2 IP Migration between the Azure network interface when HA failover occurs.


The major downside to the IP migration process is the significant amount of time required to propagate the change into the Azure environment. This may cause the business traffic to break for several minutes during the HA failover. Other critical issues may also occur, such as losing IP configuration on the Azure side during the failover.

For these reasons, FortiADC has developed a new deployment mode to avoid the issues created by the Azure API call.

:exclamation: For the Firewall 1-1 NAT configuration, the Azure API calling method is still required to migrate the IP during HA failover.

## VRRP HA using Azure Load Balancers

To avoid the issues caused by the Azure API call HA deployment mode, FortiADC has developed a new design that incorporates the use of internal and external Azure Load Balancers. The objective of this design is to eliminate the need for IP migration by making the IP configurations that are affected during an HA failover independent on each FortiADC. 
In the event of an HA failover, there are 5 configurations that will be affected:

1. Virtual Server IP
2. IP used in the VS NAT pool
3. IP used in the Firewall NAT SNAT
4. IP used as the Interface floating IP
5. IP used in the Firewall 1-1 NAT

To avoid IP migration, the IP configuration needs to be independent on each FortiADC. This means that these FortiADC IPs should not be synchronized even when HA is active. Each FortiADC should maintain its own IP configuration (for all 5 configurations listed above), both on the FortiADC and Azure side as shown in Figure 3 below.

![](https://github.com/fortinet/fortiadc-azure-ha/blob/main/figures/Figure%203%20independent%20IP%20configuration%20on%20both%20FortiADC%20side%20and%20Azure%20side.png?raw=true)

> Figure 3 independent IP configuration on both FortiADC side and Azure side

To achieve this, the FortiADC units are configured to maintain their own IP configuration. However, this is only achievable for 4 of the total 5 configurations listed above; the 5th configuration, the IP used for the Firewall 1-1 NAT, would still require using the Azure API call method for HA failover.

With FortiADC nodes. When a health check fails to one of the nodes in the HA cluster, the Azure Load Balancer will stop using the node and direct the traffic to another active node. This process only takes several seconds, which is significantly faster than migrating IP configurations.

:warning: *If you are using a virtual server IP and/or an interface floating IP, you will need to set up an Azure Load Balancer to achieve a single access endpoint.*

:exclamation: *For FortiADC version 6.2.0 or above, the VRRP HA deployment mode using Azure API call is no longer supported for virtual server IP and interface floating IP. An Azure Load Balancer is required for deploying HA for these IP configurations.*

### What is Azure Load Balancer?
Acting as the single p this new design, the independent configuration of the virtual server IP on both FortiADCs would now be using Azure Load Balancers to provide the single access point for customers in the event of HAfailover. 

The Azure Load Balancer publishes the IP address used by customers and it will balance the traffic between the oint of contact for clients, Azure Load Balancer (ALB) distributes inbound flows that arrive at the load balancer's front end to backend pool instances. For deploying the FortiADC VRRP HA, the ALB distributes the inbound traffic to the FortiADC units (the backend pool instances). These flows are determined according toconfigured load-balancing rules and health probes.

ALB provides flexibility in defining the load-balancing rules. The function of the load-balancing rule is to declare how an address and port on the front end would be mapped to the destination address and port on the backend.

In FortiADC 6.2.0, the default rule type mapping of ALB is supported, using the secondary IP of the FortiADC in the ALB backend pool configuration. As shown in Figure 4 below, with the default rule type, Azure exposes a traditional load balancing IP address scheme for ease of use (such as the VM instances' IP). 

In the Figure 4 example, when receiving business traffic, ALB changes the packet destination IP from 51.x.x.x to 10.2.0.7, which is the secondary IP of the FAD1-OutSideNic attached to the HA working node, FAD-HA-VM1.

![](https://github.com/fortinet/fortiadc-azure-ha/blob/main/figures/Figure%204%20Azure%20Load%20Balancer%20with%20FortiADC%20HA%20pair.png?raw=true)

> Figure 4 Azure Load Balancer with FortiADC HA pair

Azure Load Balancer uses health probes to monitor the FortiADCs in the backend pool by periodically sending the health check packet to all FortiADCs in the backend pool. 
The probing virtual server configured on FortiADC will respond to the health check packet from the ALB. Under known conditions in HA VRRP mode, only the probing virtual server on the working FortiADC will respond to the packet. Based on the health check result, ALB can direct the business traffic to the correct (working) FortiADC.

### Configuring the FortiADC-VMs to use Azure Load Balancer

In FortiADC 6.2.0, the new configuration object "Azure LB Backend" was introduced. This can be found via the GUI in **System** > **Azure LB Backend**. The Azure LB Backend configuration object was added to prevent the HA nodes from synchronizing in a VRRP cluster. It can also be used in the virtual server configuration screen to prevent configuration mistakes and to re-use the same IP address.

:grey_exclamation: As the Azure LB Backend configuration object would not be synchronized in HA mode, you must configure it on both the FortiADCs in the cluster. This feature is only available in the FortiADC Azure based images.

| Settings   | Guidelines  |
| ------------- | ------------- |
| Name   | Azure LB Backend Pool Name. Valid characters are A-Z, a-z, 0-9, _, and-. No spaces. After you initially save the configuration, you cannot edit the name.  |
| IP  | IP address used in the Azure LB Backend Pool. |

Using the Figure 4 example, the FAD-HA-VM1 configuration should look like this:

    FAD-HA-vm1 # show system azure-lb-backend-ip
    config system azure-lb-backend-ip
    edit "FADHaLBBackendAddrPool"
    set ip 10.2.0.7
    end


And the FAD-HA-VM2 configuration should look like this:

    FAD-HA-vm2 # show system azure-lb-backend-ip
    config system azure-lb-backend-ip
    edit "FADHaLBBackendAddrPool"
    set ip 10.2.0.5
    end



The Azure LB Backend configuration object can be used in the virtual server configuration to reference the Azure LB Backend IP as the virtual server IP.

| Settings   | Guideline |
| ------------- | ------------- |
| use-azure-lb-backend-ip   | Enable or disable the virtual server to reference Azure LB Backend IP as the virtual server IP. |
| azure-lb-backend  | Select the Azure LB Backend IP Configuration used as the virtual server IP. |

    FAD-HA-vm1 # show load-balance virtual-server probing-vs
    config load-balance virtual-server
    edit "l7-vs"
    set type l7-load-balance
    set use-azure-lb-backend-ip enable
    set azure-lb-backend FADHaLBBackendAddrPool
    set interface port1
    set port 8080
    set load-balance-profile LB_PROF_HTTP
    set load-balance-method LB_METHOD_ROUND_ROBIN
    set load-balance-pool pool
    set traffic-group default
    next
    end



The Azure LB Backend configuration is an independent configuration on all FortiADC nodes. The virtual server references the Azure LB Backend IP as the virtual Server IP. In this way, the virtual server IP is no longer a shared configuration for FAD-HA-VM1 and FAD-HA-VM2.
Therefore, when an HA failover occurs, the traffic will be directed to the new working (active) FortiADC by the ALB.


### Deploying VRRP HA using Azure load balancers via the ARM template

#### Configuring the ARM template deployment parameters
After you have successfully launched your ARM template, configure the following parameters to complete the ARM template deployment.
1. In the Azure Custom deployment where you have launched your ARM template, select the Basics tab.
2. Under the Project details, select the applicable Subscription and Resource group. The Subscription and Resource group should be the same as the ones where your license files are stored.
3. Under the Instance details, configure the following settings:


| Parameter Name   | Description |
| ------------- | ------------- |
| Region   | Select the region according to the Subscription and Resource group. |
| Subscription Id   |Apply the subscription ID. |
| Tenant Id  | Apply the tenant ID. |
| Restapp Id |  Apply the restapp ID. |
| Restapp Secret |  Apply the restapp secret. |
| Resource Name Prefix | Specify a prefix for the resources to be deployed. The names of the resources will contain the specified prefix. |
| Vm Sku | Specify the FortiADC-VM instance types. Select from the following instance types: Standard_F2s_v2/Standard_F4s_v2/Standard_F8s_v2/Standard_F16s_v2/Standard_F32s_v2. To ensure high performance, it is recommended to deploy a VM instance with at least 2 vCPUs and 8 GB memory. If you are using BYOL licensing type, specify an instance type that matches your FortiADC-VM licenses. For example, if your FortiADC-VM license supports 4 vCPUs, you can choose from the instance types that have 4 vCPUs. |
| FAD Admin Username |  Enter an administrator username for the FortiADC instances. Note: The username cannot be "admin" or "root".|
| FAD Admin Password | Enter a password for the administrator account if you have chosen password for Authentication Type. The Azure password policy requires the password to include at least 3 of the 4 from the following: Lowercase characters, Uppercase characters, Numerical digits, Special characters (Regex match [\W_]) |
| FAD Image |  Type Select BYOL or PAYG. |
| FAD Image Version | Select the image version of FortiADC-VMs. It is recommended to deploy the latest version. |
| FAD Count | Specify the number of virtual machines to be created in the HA group. The minimum is 1 and maximum is 2; the default is 2. |
|Vnet | New Or Existing Select whether to use a new or existing virtual network. |
| Vnet Resource Group | If you selected existing for Vnet New Or Existing, then specify the resource group to which the existing virtual network belongs. |
| Vnet Name |  Specify a name for the new virtual network or enter the name of the existing virtual network. |
| Vnet Address Prefix | Specify the virtual network address prefix. For example, 10.2.0.0/16. |
| Vnet Subnet1Name | Specify a name for the public-facing subnet. |
| Vnet Subnet1Prefix | Specify the prefix of the public-facing subnet. For example, 10.2.0.0/24. |
| Vnet Subnet2Name | Specify a name for the private subnet. |
| Vnet Subnet2Prefix | Specify the prefix of the private subnet. For example, 10.2.1.0/24. |
| Internal LB Frontend IP | Specify an internal load balancer front end IP. For example, 10.2.1.6.｜
| FAD1HAPort2IP| Specify the FAD1 HA Port2 IP. For example, 10.2.1.4.|
|FAD2HAPort2IP |Specify the FAD2 HA Port2 IP. For example, 10.2.1.5.|
|FAD1internal LB backendip |Specify the FAD1 internal load balancer IP. For example, 10.2.1.8.|
|FAD2internal LB backendip |Specify the FAD2 internal load balancer IP. For example, 10.2.1.9.|
|Fortiadc Ha Group Name |Specify a name for the FortiADC HA group.|
|Fortiadc Ha Group Id |Specify an ID for the FortiADC HA group. All the members in the HA group will be marked with this group ID. The minimum is 0 and the maximum is 63. |
|Storage Account Name | Specify the name of the storage account. Note: This is applicable for the serial console and if BYOL is selected as the FAD Image Type. |
|Storage License Container Name| Enter the name of the containers where the license files are stored. Note: This is applicable only if BYOL is selected as the FAD Image Type. |
|Storage Licensefile1 | Enter one of the names of the two licenses you have uploaded into the storage license container. For example, FADXXXlic.|
|Storage Licensefile2 | Enter one of the names of the two licenses you have uploaded into the storage license container. For example, FADXXXlic. |

4.  Click Review + create.
5. Check the resource group with all the deployment resources. The following lists the deployed resources in the existing virtual network.

| Deployed Resource | Description |
| ------------- | ------------- |
| FAD-HA-example-vm1 | FortiADC in the HA group.|
| FAD-HA-example-vm2 | FortiADC in the HA group. |
|FAD-HA-example-external-nic1 | External interface of the FAD-HA-example-vm1 for external access.|
| FAD-HA-example-internal-nic1 | Internal interface of the FAD-HA-example-vm1 for internal access to the protected server. This is also used as the HA VRRP Unicast IP. |
| FAD-HA-example-external-nic2 | External interface of the FAD-HA-example-vm2 for external access. | 
| FAD-HA-example-internal-nic2 | Internal interface of the FAD-HA-example-vm2 for internal access to the protected server. This is also used as the HA VRRP Unicast IP. |
| FAD-HA-example-loadbalance-internal |  Internal Azure Load Balancer. This is used in the FortiADC L4 virtual server topology. For more information, see Example: FortiADC HA with L4 VS Topology using Azure Load Balancer. |
| FAD-HA-example-loadbalance-external | External Azure Load Balancer. This provides the single access endpoint for the FortiADC virtual servers. |
| FAD-HA-example-nicPublic-IP1 | Provides public access to the FAD-HA-example-vm1.|
| FAD-HA-example-nicPublic-IP2 |  Provides public access to the FAD-HA-example-vm2. |
| FAD-HA-example-loadbalance-IP |  External access to the ALB FAD-HA-example-loadbalance-external. |
| FAD-HA-exampleRouteTable-FadcHAInsideSubnet | Routing table for L4 virtual server topology. For more information, see Example: FortiADC HA with L4 VS Topology using Azure Load Balancer. |
| FAD-HA-example-availabilitySet | Provided for redundancy and availability. |
| FAD-HA-example-securityGroup | Access rules for the external subnet. | 
| FAD-HA-example-securityGroup2 | Access rules for the internal subnet. |
|FortiADC-vnet-example |  Virtual network where the FortiADCs are located.

6. If you are using an existing virtual network, you will need to manually associate the subnet2 to route the table for the FAD-HA-exampleRouteTable-FadcHAInsideSubnet.
a. From the list of deployed resources in the existing virtual network, select **FAD-HA-exampleRouteTable-FadcHAInsideSubnet.**
b. In the **Settings** section, select **Subnets**.
c. Click **+Associate**.
![](https://github.com/fortinet/fortiadc-azure-ha/blob/main/figures/associate_subnet_for%20existing_network.png?raw=true)

7. Check the FortiADC console to ensure the license (BYOL) is installed and the ha init is done.
a. Select the FortiADC-VM in the HA group. For example, the FAD-HA-example-vm1.
b. In the **Support + troubleshooting** section, select **Serial console**.
![](https://github.com/fortinet/fortiadc-azure-ha/blob/main/figures/ha_init_done.png?raw=true)

### Example: FortiADC HA with L7 VS Topology using Azure Load Balancer
![](https://github.com/fortinet/fortiadc-azure-ha/blob/main/figures/l7_vs_azure_ha_topology.png?raw=true)

After you have deployed the ARM template, you will see the following objects in your resource group (this list
includes only a small subset of all the objects you will see):
- An external Azure Load Balancer (FAD-HA-loadbalance-external)
- Azure Virtual Network (FadcHAOutsideSubnet/FadcHAInsideSubnet)
- 2 FortiADC-VMs (FAD-HA-VM1 and FAD-HA-VM2) The FortiADC nodes are already configured with HA VRRP.

#### Creating an L7 virtual server to allow user access to protected resources from the internet

Follow the steps below to create an L7 virtual server to let the user access the protected resources from the internet.
In this example, we assume the following:
- You have a web server available.
- The web server is configured to use the VNET named FadcHAInsideSubnet.
In the example below, we will show you how to enable secure access from internet clients to a published Apache Tomcat server.

1. Prepare your protected resource (the web server) and ensure it is connected to the VNET, FadcHAInsideSubnet. For this scenario, we have prepared an Apache Tomcat server.

2. Check the Frontend IP configuration of the ALB, FAD-HA-loadbalance-external. This IP configuration, such as 51.x.x.x, is the Public IP for users to access the protected resource.
![](https://github.com/fortinet/fortiadc-azure-ha/blob/main/figures/azure-lb-fronted-ip.png?raw=true)

3. Check the Backend pools of the ALB, FAD-HA-loadbalance-external. You will see both the FortiADC-VMs are in the FADHaLBBackendAddrPool with IP allocated from FadcOutSideSubnet.
![](https://github.com/fortinet/fortiadc-azure-ha/blob/main/figures/azure-lb-backend-ip.png?raw=true)

4. Check the Health Probe in ALB FAD-HA-loadbalance-external.
When receiving business traffic from the user, FAD-HA-loadbalance-external directs the business traffic to the HA VRRP working FortiADC-VM, FAD-HA-VM1. ALB determines which FortiADC-VM is the working node by sending the health check packet to both FortiADC-VMs to see which will respond to the health check packet.
![](https://github.com/fortinet/fortiadc-azure-ha/blob/main/figures/azure-lb-health-probe.png?raw=true)
You can see there are 2 TCP Health Probes with port 8080 and 10443 respectively.
5. To enable the health check flow to work, the probing virtual server needs to be configured to respond to the health check packet on the FortiADC side.
a. Configure the backend pool IP information of the FAD-HA-loadbalance-external on both FortiADC-VMs.
![](https://github.com/fortinet/fortiadc-azure-ha/blob/main/figures/adc-lb-backend-ip.png?raw=true)
![](https://github.com/fortinet/fortiadc-azure-ha/blob/main/figures/adc-lb-backend-ip2.png?raw=true)
b. Configure the probing VS which references the ext-FADHaLBBackendAddrPool-1 as the virtual server
IP. Set the probing VS with port 8080 and profile LB_PROF_TCP according to the tcpprobe as defined in the Health Probes of FAD-HA-loadbalance-external.
:warning: For FortiADC version 6.2.1 or later, LB_PROF_TCP is deprecated. Please use LB_PROF_L7_TCP instead.
![](https://github.com/fortinet/fortiadc-azure-ha/blob/main/figures/probing_vs.png?raw=true)
Now, we have the probing VS using FADHaLBBackendAddrPOOL IP on both FortiADC-VMs.
![](https://github.com/fortinet/fortiadc-azure-ha/blob/main/figures/ext-probing-vs-adc1.png?raw=true)
![](https://github.com/fortinet/fortiadc-azure-ha/blob/main/figures/ext-probing-vs-adc2.png?raw=true)
6. Create the L7 virtual server with port 443 and LB_PROF_HTTPS profile on the FortiADC-VM1. This virtual server is going to serve the HTTPS service.
We only support the ALB default route type. We also reference the azure loadbalancer backend IP in the L7 virtual server.
![](https://github.com/fortinet/fortiadc-azure-ha/blob/main/figures/l7-vs-https.png?raw=true)

7. Check the Load balancing rules in FAD-HA-loadbalance-external. Change the LBRule1 to use tcpprobe as the Heath probe.
![](https://github.com/fortinet/fortiadc-azure-ha/blob/main/figures/l7-vs-https-lb-rule.png?raw=true)

8. Try to connect to https://51.x.x.x to get apache tomcat service
![](https://github.com/fortinet/fortiadc-azure-ha/blob/main/figures/l7-vs-https-tomcat.png?raw=true)

### Example: FortiADC HA with L4 VS Topology using Azure Load Balancer
To be continued...


## Support

Please contact your Fortinet representation for any comments, questions, considerations, and/or concerns.

## License

[License](./LICENSE) © Fortinet Technologies. All rights reserved.
