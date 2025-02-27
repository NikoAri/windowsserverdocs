---
title: Deploy a Software Defined Network Infrastructure Using Scripts
description: This topic covers how to deploy a Microsoft Software Defined Network (SDN) infrastructure using scripts in Windows Server.
ms.topic: how-to
ms.assetid: 5ba5bb37-ece0-45cb-971b-f7149f658d19
ms.author: roharwoo
author: robinharwood
ms.date: 08/13/2024
---

# Deploy a Software Defined Network infrastructure using scripts

>

In this topic, you deploy a Microsoft Software Defined Network (SDN) infrastructure using scripts. The infrastructure includes a highly available (HA) network controller, an HA Software Load Balancer (SLB)/MUX, virtual networks, and associated Access Control Lists (ACLs). Additionally, another script deploys a tenant workload for you to validate your SDN infrastructure.

If you want your tenant workloads to communicate outside their virtual networks, you can set up SLB NAT rules, Site-to-Site Gateway tunnels, or Layer-3 Forwarding to route between virtual and physical workloads.

You can also deploy an SDN infrastructure using Virtual Machine Manager (VMM). For more information, see [Set up a Software Defined Network (SDN) infrastructure in the VMM fabric](/system-center/vmm/deploy-sdn).

## Predeployment

> [!IMPORTANT]
> Before you begin deployment, you must plan and configure your hosts and physical network infrastructure. For more information, see [Plan a Software Defined Network Infrastructure](/azure/azure-local/concepts/plan-software-defined-networking-infrastructure?context=/windows-server/context/windows-server-edge-networking).

All Hyper-V hosts must have Windows Server 2019 or 2016 installed.

## Deployment steps
Start by configuring the Hyper-V host's (physical servers) Hyper-V virtual switch and  IP address assignment. Any storage type that is compatible with Hyper-V, shared, or local may be used.

### Install host networking

1. Install the latest network drivers available for your NIC hardware.

2. Install the Hyper-V role on all hosts. For more information, see [Install the Hyper-V role on Windows Server](/windows-server/virtualization/hyper-v/get-started/install-the-hyper-v-role-on-windows-server).

   ```PowerShell
   Install-WindowsFeature -Name Hyper-V -ComputerName <computer_name> -IncludeManagementTools -Restart
   ```

3. Create the Hyper-V virtual switch.<p>Use the same switch name for all hosts, for example, **sdnSwitch**. Configure at least one network adapter or, if using SET, configure at least two network adapters. Maximum inbound spreading occurs when using two NICs.

   ```PowerShell
   New-VMSwitch "<switch name>" -NetAdapterName "<NetAdapter1>" [, "<NetAdapter2>" -EnableEmbeddedTeaming $True] -AllowManagementOS $True
   ```

   > [!TIP]
   > You  can skip steps 4 and 5 if you have separate Management NICs.

4. Refer to the planning topic [Plan a Software Defined Network Infrastructure](/azure/azure-local/concepts/plan-software-defined-networking-infrastructure?context=/windows-server/context/windows-server-edge-networking) and work with your network administrator to obtain the VLAN ID of the Management VLAN. Attach the Management vNIC of the newly created Virtual Switch to the Management VLAN. This step can be omitted if your environment doesn't use VLAN tags.

   ```PowerShell
   Set-VMNetworkAdapterIsolation -ManagementOS -IsolationMode Vlan -DefaultIsolationID <Management VLAN> -AllowUntaggedTraffic $True
   ```

5. Refer to the planning topic [Plan a Software Defined Network Infrastructure](/azure/azure-local/concepts/plan-software-defined-networking-infrastructure?context=/windows-server/context/windows-server-edge-networking) and work with your network administrator to use either DHCP or static IP assignments to assign an IP address to the Management vNIC of the newly created vSwitch. The following example shows how to create a static IP address and assign it to the Management vNIC of the vSwitch:

   ```PowerShell
   New-NetIPAddress -InterfaceAlias "vEthernet (<switch name>)" -IPAddress <IP> -DefaultGateway <Gateway IP> -AddressFamily IPv4 -PrefixLength <Length of Subnet Mask - for example: 24>
   ```

6. [Optional] Deploy a virtual machine to host Active Directory Domain Services [Install Active Directory Domain Services (Level 100)](../../../identity/ad-ds/deploy/install-active-directory-domain-services--level-100-.md) and a DNS Server.

    a. Connect the Active Directory/DNS Server virtual machine to the Management VLAN:

    ```PowerShell
    Set-VMNetworkAdapterIsolation -VMName "<VM Name>" -Access -VlanId <Management VLAN> -AllowUntaggedTraffic $True
    ```

   b. Install Active Directory Domain Services and DNS.

   > [!NOTE]
   > The network controller supports both Kerberos and X.509 certificates for authentication. This guide uses both authentication mechanisms for different purposes (although only one is required).

7. Join all Hyper-V hosts to the domain. Ensure the DNS server entry for the network adapter that has an IP address assigned to the Management network points to a DNS server that can resolve the domain name.

   ```PowerShell
   Set-DnsClientServerAddress -InterfaceAlias "vEthernet (<switch name>)" -ServerAddresses <DNS Server IP>
   ```

   a. Right-click **Start**, select **System**, and then select **Change Settings**.
 **Change Settings**.
   b. Select **Change**.
   c. Select **Domain** and specify the domain name.
   d. Select **OK**.
   e. Type the user name and password credentials when prompted.
   f. Restart the server.

### Validation

Use the following steps to validate that host networking is set up correctly.

1. Ensure the VM Switch was created successfully:

   ```PowerShell
   Get-VMSwitch "<switch name>"
   ```

2. Verify that the Management vNIC on the VM Switch is connected to the Management VLAN:

   > [!NOTE]
   > Relevant only if Management and Tenant traffic share the same NIC.

   ```PowerShell
   Get-VMNetworkAdapterIsolation -ManagementOS
   ```

3. Validate all Hyper-V hosts and external management resources, for example, DNS servers. Ensure they're accessible via ping using their Management IP address and/or fully qualified domain name (FQDN).

   ```PowerShell
   ping <Hyper-V Host IP>
   ping <Hyper-V Host FQDN>
   ```

4. Run the following command on the deployment host and specify the FQDN of each Hyper-V host to ensure the Kerberos credentials used provides access to all the servers.

   ```PowerShell
   winrm id -r:<Hyper-V Host FQDN>
   ```

### Run SDN Express scripts

1. Go to the [Microsoft SDN GitHub Repository](https://github.com/Microsoft/SDN.git) for the installation files.

2. Download the installation files from the repository to the designated deployment computer. Click **Clone or download** and then click **Download ZIP**.

   > [!NOTE]
   > The designated deployment computer must be running Windows Server 2016 or later.

3. Expand the zip file and copy the **SDNExpress** folder to the deployment computer's `C:\` folder.

4. Share the `C:\SDNExpress` folder as "**SDNExpress**" with permission for **Everyone** to **Read/Write**.

5. Navigate to the `C:\SDNExpress` folder.<p>You see the following folders:

   | Folder Name | Description |
   |--|--|
   | AgentConf | Holds fresh copies of OVSDB schemas used by the SDN Host Agent on each Windows Server 2016 Hyper-V host to program network policy. |
   | Certs | Temporary shared location for the NC certificate file. |
   | Images | Empty, place your Windows Server 2016 vhdx image here |
   | Tools | Utilities for troubleshooting and debugging. Copied to the hosts and virtual machines.  We recommend you place Network Monitor or Wireshark here so it is available if needed. |
   | Scripts | Deployment scripts.<p> - **SDNExpress.ps1**<br>Deploys and configures the fabric, including the Network controller virtual machines, SLB Mux virtual machines, gateway pool(s), and the HNV gateway virtual machine(s) corresponding to the pool(s).<br /> - **FabricConfig.psd1**<br>A configuration file template for the SDNExpress script. You'll customize this for your environment.<br /> - **SDNExpressTenant.ps1**<br>Deploys a sample tenant workload on a virtual network with a load balanced VIP.<br>Also provisions one or more network connections (IPSec S2S VPN, GRE, L3) on the service provider edge gateways which are connected to the previously created tenant workload. The IPSec and GRE gateways are available for connectivity over the corresponding VIP IP Address, and the L3 forwarding gateway over the corresponding address pool.<br>This script can be used to delete  the corresponding configuration with an Undo option as well.<br /> - **TenantConfig.psd1**<br>A template configuration file for tenant workload and S2S gateway configuration.<br /> - **SDNExpressUndo.ps1**<br>Cleans up the fabric environment and resets it to a starting state.<br /> - **SDNExpressEnterpriseExample.ps1**<br>Provisions one or more enterprise site environments with one Remote Access Gateway, and (optionally) one corresponding enterprise virtual machine per site. The IPSec or GRE enterprise gateways connects to the corresponding VIP IP address of the service provider gateway to establish the S2S tunnels. The L3 Forwarding Gateway connects over the corresponding Peer IP Address. <br> This script can be used to delete the corresponding configuration with an Undo option as well.<br /> - **EnterpriseConfig.psd1**<br>A template configuration file for the Enterprise site-to-site gateway and Client VM configuration. |
   | TenantApps | Files used to deploy example tenant workloads. |

6. Verify the Windows Server 2016 VHDX file is in the **Images** folder.

7. Customize the SDNExpress\scripts\FabricConfig.psd1 file by changing the **<< Replace >>** tags with specific values to fit your lab infrastructure including host names, domain names, usernames and passwords, and network information for the networks listed in the Planning Network topic.

8. Create a Host A record in DNS for the NetworkControllerRestName (FQDN) and NetworkControllerRestIP.

9. Run the script as a user with domain administrator credentials:

   ```PowerShell
   SDNExpress\scripts\SDNExpress.ps1 -ConfigurationDataFile FabricConfig.psd1 -Verbose
   ```

10. To undo all operations, run the following command:

   ```PowerShell
    SDNExpress\scripts\SDNExpressUndo.ps1 -ConfigurationDataFile FabricConfig.psd1 -Verbose
   ```

#### Validation

Assuming that the SDN Express script ran to completion without reporting any errors, you can perform the following step to ensure the fabric resources have been deployed correctly and are available for tenant deployment.

Use [Diagnostic Tools](../troubleshoot/troubleshoot-windows-server-software-defined-networking-stack.md) to ensure there are no errors on any fabric resources in the network controller.

   ```PowerShell
   Debug-NetworkControllerConfigurationState -NetworkController <FQDN of Network Controller Rest Name>
   ```

### Deploy a sample tenant workload with the software load balancer

Now that fabric resources have been deployed, you can validate your SDN deployment end-to-end by deploying a sample tenant workload. This tenant workload consists of two virtual subnets (web tier and database tier) protected via Access Control List (ACL) rules using the SDN distributed firewall. The web tier's virtual subnet is accessible through the SLB/MUX using a Virtual IP (VIP) address. The script automatically deploys two web tier virtual machines and one database tier virtual machine and connects these to the virtual subnets.

1. Customize the SDNExpress\scripts\TenantConfig.psd1 file by changing the **<< Replace >>** tags with specific values (for example: VHD image name, network controller REST name, vSwitch Name, etc. as previously defined in the FabricConfig.psd1 file)

2. Run the script. For example:

   ```PowerShell
   SDNExpress\scripts\SDNExpressTenant.ps1 -ConfigurationDataFile TenantConfig.psd1 -Verbose
   ```

3. To undo the configuration, run the same script with the **undo** parameter. For example:

   ```PowerShell
   SDNExpress\scripts\SDNExpressTenant.ps1 -Undo -ConfigurationDataFile TenantConfig.psd1 -Verbose
   ```

#### Validation

To validate that the tenant deployment was successful, do the following:

1. Log in to the database tier virtual machine and try to ping the IP address of one of the web tier virtual machines (ensure Windows Firewall is turned off in web tier virtual machines).

2. Check the network controller tenant resources for any errors. Run the following from any Hyper-V host with Layer-3 connectivity to the network controller:

   ```PowerShell
   Debug-NetworkControllerConfigurationState -NetworkController <FQDN of Network Controller REST Name>
   ```

3. To verify that the load balancer is running correctly, run the following from any Hyper-V host:

   ```PowerShell
   wget <VIP IP address>/unique.htm -disablekeepalive -usebasicparsing
   ```

   where `<VIP IP address>` is the web tier VIP IP address you configured in the TenantConfig.psd1 file.

   > [!TIP]
   > Search for the `VIPIP` variable in TenantConfig.psd1.

   Run this multiple times to see the load balancer switch between the available DIPs. You can also observe this behavior using a web browser. Browse to `<VIP IP address>/unique.htm`. Close the browser and open a new instance and browse again. You'll see the blue page and the green page alternate, except when the browser caches the page before the cache times out.
