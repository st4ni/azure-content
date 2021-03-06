<properties
   pageTitle="Implementing a Hybrid Network Architecture with Azure ExpressRoute | Blueprint | Microsoft Azure"
   description="How to implement a secure site-to-site network architecture that spans an Azure virtual network and an on-premises network connected by using Azure ExpressRoute."
   services=""
   documentationCenter="na"
   authors="telmosampaio"
   manager="christb"
   editor=""
   tags=""/>

<tags
   ms.service="guidance"
   ms.devlang="na"
   ms.topic="article"
   ms.tgt_pltfrm="na"
   ms.workload="na"
   ms.date="04/14/2016"
   ms.author="telmos"/>

# Azure Blueprints: Implementing a Hybrid Network Architecture with Azure ExpressRoute

This article describes best practices for connecting an on-premises network to virtual networks on Azure by using ExpressRoute. ExpressRoute connections are made using a private dedicated connection through a third-party connectivity provider. The private connection extends your on-premises network into Azure providing access to your own IaaS infrastructure in Azure, public endpoints used in PaaS services, and Office365 SaaS services. This document focuses on using ExpressRoute to connect to a single Azure virtual network (VNet) using what is called private peering.

> [AZURE.NOTE] Azure has two different deployment models: [Resource Manager][resource-manager-overview] and classic. This blueprint uses Resource Manager, which Microsoft recommends for new deployments.

Typical use cases for this architecture include:

- Hybrid applications where workloads are distributed between an on-premises network and Azure.

- Applications running large-scale, mission-critical workloads that require a high degree of scalability.

- Large-scale backup and restore facilities for data that must be saved off-site.

- Handling Big Data workloads.

- Using Azure as a disaster-recovery site.

> [AZURE.NOTE] The [ExpressRoute technical overview][expressroute-technical-overview] provides an introduction to ExpressRoute.

## Architecture blueprint

The following diagram highlights the important components in this architecture:

![IaaS: hybrid-expressroute](./media/guidance-hybrid-network-expressroute/figure1.png)

- **Azure Virtual Networks (VNets).** Each VNet can span multiple tiers, and tiers can be segmented using subnets in each VNet  and/or network security groups (NSGs). The VNets could reside in [multiple subscriptions][link-vnet-to-expressroute] in the same [geo-political region][expressroute-locations]. Connections are performed by using **private peering** utilizing addresses which are private to the VNet.

- **Azure public services.** These are Azure services that can be utilized within a hybrid application. These services are also available across the Internet, but accessing them via an ExpressRoute circuit provides low latency and more predictable performance since traffic does not go through the Internet. Connections are performed by using **public peering**, and traffic is routed to [IP address ranges published by Microsoft][datacenter-ip-ranges]. The ExpressRoute circuit performs a NAT translation of on-premises traffic to an endpoint in one of these ranges. Connections over public peering can only be initiated from on-premises.

- **Office 365 services.** These are the publicly available Office 365 applications and services provided by Microsoft. Connections are performed by using **Microsoft peering**, with addresses that are either owned by your organization or supplied by your connectivity provider. 

- **On-premises corporate network.** This is a network of computers and devices, connected through a private local-area network running within an organization.

- **Local edge routers.** These are routers that connect the on-premises network to the circuit managed by the provider.

- **Microsoft edge routers.** These are two routers in an active-active highly available configuration. These routers enable a connectivity provider to connect their circuits directly to their datacenter.

- **ExpressRoute circuit.** This is a layer 2 or layer 3 circuit supplied by the connectivity provider that joins the on-premises network with Azure through the edge routers. The circuit uses the hardware infrastructure managed by the connectivity provider.

- **Connectivity providers.** These are companies that provide a connection either using layer 2 or layer 3 connectivity between your datacenter and an Azure datacenter.

> [AZURE.NOTE] ExpressRoute connectivity providers connect your datacenter to Azure in one of three different ways:

> - **Co-located at a cloud exchange.** If you're co-located in a facility with a cloud exchange, you can order virtual cross-connections to the Microsoft cloud through the co-location provider’s Ethernet exchange. Co-location providers can offer either Layer 2 cross-connections, or managed Layer 3 cross-connections between your infrastructure in the co-location facility and the Microsoft cloud..
>
> - **Point-to-point Ethernet connections.** You can connect your on-premises datacenters/offices to the Microsoft cloud through point-to-point Ethernet links. Point-to-point Ethernet providers can offer Layer 2 connections, or managed Layer 3 connections between your site and the Microsoft cloud.
>
> - **Any-to-any (IPVPN) networks.** You can integrate your WAN with the Microsoft cloud. IPVPN providers (typically MPLS VPN) offer any-to-any connectivity between your branch offices and datacenters. The Microsoft cloud can be interconnected to your WAN to make it look just like any other branch office. WAN providers typically offer managed Layer 3 connectivity.
>
> To obtain a list of connectivity providers available at your location, use the following Azure PowerShell command:
> ```
> Get-AzureRmExpressRouteServiceProvider
> ```
> For more information about connectivity providers, see [ExpressRoute circuits and routing domains][connectivity-providers].

## Implementing this architecture

The following high-level steps outline a process for implementing this architecture. Detailed examples using Azure PowerShell commands are described [later in this document][sample-script]. Note that this process assumes that you have already created a VNet for hosting the cloud application, that you have created the on-premises network, and that your organization has met the [ExpressRoute prerequiste requirements][expressroute-prereqs] for connecting to the Azure. 

1. Create an ExpressRoute circuit by using the following command:

    ```powershell
     New-AzureRmExpressRouteCircuit -Name <<circuit-name>> -ResourceGroupName <<resource-group>> -Location <<location>> -SkuTier <<sku-tier>> `
        -SkuFamily <<sku-family>> -ServiceProviderName <<service-provider-name>> -PeeringLocation <<peering-location>> -BandwidthInMbps <<bandwidth-in-mbps>>
    ```
2. Arrange for the ExpressRoute circuit to be provisioned. If you're using a level 2 connection, go to item 3. If you're using a level 3 connection, follow the steps below.

	1. Send the `ServiceKey` for the new circuit to the service provider, together with the address of a /27 subnet that is outside the range of you on-premises network(s) and Azure VNet(s).

		> [AZURE.NOTE] The service provider may provide an online portal for you to supply this information.

	2. Wait for the provider to provision the circuit. You can verify the provisioning state of a circuit by using the following PowerShell command:

        ```powershell
        Get-AzureRmExpressRouteCircuit -Name <<circuit-name>> -ResourceGroupName <<resource-group>>
        ```

		The `Provisioning state` field in the `Service Provider` section of the output will change from `NotProvisioned` to `Provisioned` when the circuit is ready.

		>[AZURE.NOTE]If you're using a level 3 connection, the provider should configure and manage routing for you; you provide the information necessary to enable the provider to implement the appropriate routes.

3. Arrange for the ExpressRoute circuit to be provisioned for a level 2 connection:

	1. Send the `ServiceKey` for the new circuit to the service provider.

	2. Reserve several blocks of IP addresses to configure routing between your network and the Microsoft edge routers. Each peering requires two /30 subnets. For example, if you're implementing a private peering to a VNet and a public peering for accessing Azure services, you will require four /30 subnets. This is for availability purposes; one subnet provides a primary circuit while the other acts as a secondary circuit. The IP prefixes for these subnets cannot overlap with the IP prefixes used by your VNet or on-premises networks. For details, see [Create an ExpressRoute circuit][create-expressroute-circuit].	

	3. Wait for the provider to provision the circuit.

	4. Configure routing for the ExpressRoute circuit.
	
	>[AZURE.NOTE]If you're using a level 2 connection, you will most likely be responsible for configuring routing yourself, using the /30 subnet addresses that you reserved. See [Create and modify routing for an ExpressRoute circuit][configure-expressroute-routing] for details. Use the following PowerShell commands to add a network peering for routing traffic:

	>```powershell
	>Set-AzureRmExpressRouteCircuitPeeringConfig -Name <<peering-name>> -Circuit <<circuit-name>> -PeeringType <<peering-type>> -PeerASN <<peer-asn>> -PrimaryPeerAddressPrefix <<primary-peer-address-prefix>> -SecondaryPeerAddressPrefix <<secondary-peer-address-prefix>> -VlanId <<vlan-id>>

	>Set-AzureRmExpressRouteCircuit -ExpressRouteCircuit <<circuit-name>>
	>```

	>The `PeeringType` parameter can be one of `AzurePublicPeering`, `MicrosoftPeering`, or `AzurePrivatePeering`.
	>
	> For more information about primary and secondary peering addresses, see [Availability](#availability).

	Depending on your requirements, you may need to perform the following operations:

	- Configure private peering for connecting between on-premises services and components running in the VNet.

	- Configure public peering for connecting between on-premises services and Azure public services.

	- Configure Microsoft peering for connecting between on-premises services and Office 365 services.


4. [Link your private VNet(s) in the cloud to the ExpressRoute circuit][link-vnet-to-expressroute]. Use the following PowerShell commands:

	```
	$circuit = Get-AzureRmExpressRouteCircuit -Name <<circuit-name>> -ResourceGroupName <<resource-group>>
	$gw = Get-AzureRmVirtualNetworkGateway -Name <<gateway-name>> -ResourceGroupName <<resource-group>>
	New-AzureRmVirtualNetworkGatewayConnection -Name <<connection-name>> -ResourceGroupName <<resource-group>> -Location <<location> -VirtualNetworkGateway1 $gw -PeerId $circuit.Id -ConnectionType ExpressRoute
	```

>[AZURE.NOTE] For more information about this process, see [ExpressRoute workflows for circuit provisioning and circuit states][ExpressRoute-provisioning].

Note the following points:

- ExpressRoute uses the Border Gateway Protocol (BGP) for exchanging routing information between your network and Azure.

- You can connect multiple VNets located in different regions to the same ExpressRoute circuit as long as all VNets and the ExpressRoute circuit are located within the same geopolitical region.

## Availability

You can configure high availability for your Azure connection in different ways, depending on the type of provider you use, and the number of ExpressRoute circuits and VPN gateway connection you're willing to configure. The points below summarize your availability options:

- ExpressRoute does not support router redundancy protocols such as HSRP and VRRP to implement high availability. Instead, it uses a redundant pair of BGP sessions per peering. To facilitate highly-available connections to your network, Microsoft Azure provisions you with two redundant ports on two routers (part of the Microsoft edge) in an active-active configuration.

- If you're using a level 2 connection, deploy redundant routers in your on-premises network in an active-active configuration. Connect the primary circuit to one router, and the secondary circuit to the other. This will give you a highly available connection at both ends of the connection. This is necessary if you require the ExpressRoute SLA. See [SLA for Azure ExpressRoute][sla-for-expressroute] for details.

	The following diagram shows a configuration with redundant on-premises routers connected to the primary and secondary circuits. Each circuit handles the traffic for a public peering and a private peering (each peering is designated a pair of /30 address spaces, as described in the previous section).

	![IaaS: hybrid-expressroute](./media/guidance-hybrid-network-expressroute/figure2.png)

- If you're using a level 3 connection, verify that it provides redundant BGP sessions that handle availability for you.

- Virtual networks can be connected to multiple ExpressRoute circuits and each  circuits can be supplied by different service providers. This strategy provides additional high-availability and disaster recovery capabilities.

- Configure a Site-to-Site VPN as a failover path for ExpressRoute. This is only applicable to private peering. For Azure and Office 365 services, the Internet is the only failover path.

## Security

You can configure security options for your Azure connection in different ways, depending on your security concerns and compliance needs. The points below summarize your security options.

- ExpressRoute operates in layer 3. Threats in the Application layer can be prevented by using a Network Security Appliance which restricts traffic to legitimate resources. Additionally, ExpressRoute connections can only be initiated from on-premises; this helps to prevent a rogue service from accessing and compromising on-premises data.

- To maximize security, add network security appliances between the on-premises network and the provider edge routers. This will help to restrict the inflow of unauthorized traffic from the VNet:

	![IaaS: hybrid-expressroute-firewalls](./media/guidance-hybrid-network-expressroute/figure3.png)

- For auditing or compliance purposes, it may be necessary to prohibit direct access from components running in the VNet to the Internet. In this situation, Internet traffic should be redirected back through a gateway or proxy running  on-premises where it can be audited. The gateway can be configured to block unauthorized traffic flowing out, and filter potentially malicious inbound traffic.

	![IaaS: hybrid-expressroute-proxy](./media/guidance-hybrid-network-expressroute/figure4.png)

- By default, Azure VMs expose endpoints used for providing login access for management purposes - RDP and Remote Powershell for Windows VMs, and SSH for Linux-based VMs when deployed through the Azure portal. To maximize security, do not enable a public IP address for your VMs, and use NSGs to ensure that these VMs aren't publicly accessible. VMs should only be available using the internal IP address. These addresses can be made accessible through the ExpressRoute network, enabling on-premises DevOps staff to perform any necessary configuration or maintenance.

- If you must expose management endpoints for VMs to an external network, use NSGs and/or access control lists to restrict the visibility of these ports to a whitelist of IP addresses or networks.

## Scalability

ExpressRoute circuits provide a high bandwidth path between networks. Generally, the higher the bandwidth the greater the cost. If your connectivity provider offers different bandwidths, you may be able to scale by selecting the most appropriate bandwidth for a given workload and time.

- To minimize financial costs, start with the smallest estimated bandwidth that you expect to require. You can change the bandwidth of an existing circuit (if your connectivity provider supports this feature) by using the following command:

	```powershell
	$ckt = Get-AzureRmExpressRouteCircuit -Name <<circuit-name>> -ResourceGroupName <<resource-group>>

    $ckt.ServiceProviderProperties.BandwidthInMbps = <<bandwidth-in-mbps>>

    Set-AzureRmExpressRouteCircuit -ExpressRouteCircuit $ckt
	```

	You can increase the bandwidth without loss of connectivity. Downgrading the bandwidth will result in disruption in connectivity. You have to delete the circuit and recreate it with the new configuration.

- Start with the standard SKU of ExpressRoute, and upgrade to ExpressRoute Premium only when required. If necessary, you can also change the pricing plan (metered or unlimited). Switch the SKU and pricing plan by using the following commands (the `Sku.Tier` property can be `Standard` or `Premium`; the `Sku.Name` property can be `MeteredData` or `UnlimitedData`):

	```powershell
	$ckt = Get-AzureRmExpressRouteCircuit -Name <<circuit-name>> -ResourceGroupName <<resource-group>>

    $ckt.Sku.Tier = "Premium"
    $ckt.Sku.Family = "MeteredData"
	$ckt.Sku.Name = "Premium_MeteredData"

    Set-AzureRmExpressRouteCircuit -ExpressRouteCircuit $ckt
	```

	>[AZURE.IMPORTANT] Make sure the `Sku.Name` property matches the `Sku.Tier` and `Sku.Family`. If you change the family and tier, but not the name, your connection will be disabled.

	You can upgrade the SKU without disruption, but you cannot switch from the unlimited pricing plan to metered. When downgrading the SKU, your bandwidth consumption must remain within the default limit of the standard SKU.

	> [AZURE.NOTE] ExpressRoute offers two pricing plans to customers, based on metering or unlimited data. See [ExpressRoute pricing][expressroute-pricing] for details. Charges vary according to circuit bandwidth. Available bandwidth will likely vary from provider to provider. Use the `Get-AzureRmExpressRouteServiceProvider` cmdlet to see the providers available in your region and the bandwidths that they offer.
	>
	> A single ExpressRoute circuit can support a number of peerings and VNet links. See [ExpressRoute limits][expressroute-limits] for more information.
	>
	> For an extra charge, ExpressRoute Premium Add-on provides:
	>
	> - Up to 10,000 routes per circuit. Without ExpressRoute Premium Add-on, the limit is 4,000 routes per circuit.
	>
	> - Global connectivity, enabling an ExpressRoute circuit located anywhere in the world to access resources in any region rather than just regions in the same continent.
	>
	> - An increase in the number of VNet links per circuit from 10 to a larger limit, depending on the bandwidth of the circuit.

- ExpressRoute circuits are designed to allow temporary network bursts up to two times the bandwidth limit that you procured for no additional cost. This is achieved by using redundant links. However, not all connectivity providers support this feature; verify that your connectivity provider enables this feature before depending on it.

## Monitoring

At this time, your monitoring options are based on the tools available through your connection provider and on-premises network.

## Troubleshooting

If a previously functioning ExpressRoute circuit now fails to connect, in the absence of any configuration changes on-premises or within your private VNet, you may need to contact the connectivity provider and work with them to correct the issue. You can also use the following Azure PowerShell commands to perform some limited checking and help determine where problems might lie:

- Verify that the circuit has been provisioned:

	```powershell
	Get-AzureRmExpressRouteCircuit -Name <<circuit-name>> -ResourceGroupName <<resource-group>>
	```

	The output of this command shows several properties for your circuit, including `ProvisioningState`, `CircuitProvisioningState`, and `ServiceProviderProvisioningState` as shown below.
	
		ProvisioningState                : Succeeded
		Sku                              : {
		                                     "Name": "Standard_MeteredData",
		                                     "Tier": "Standard",
		                                     "Family": "MeteredData"
		                                   }
		CircuitProvisioningState         : Enabled
		ServiceProviderProvisioningState : NotProvisioned

- If the `ProvisioningState` is not set to `Suceeded` after you tried to create a new circuit, remove the circuit by using the command below and try to create it again.

	```powershell
	Remove-AzureRmExpressRouteCircuit -Name <<circuit-name>> -ResourceGroupName <<resource-group>>
	```
- If your provider had already provisioned the circuit, and the `ProvisioningState` is set to `Failed`, or the `CircuitProvisioningState` is not `Enabled`, contact your provider for further assistance.

## Deploying the sample solution

The [Azure PowerShell][azure-powershell] commands in this section show how to connect an on-premises network to an Azure VNet by using an ExpressRoute connection. This script assumes that you're using a level 3 connection.

To use the script below, execute the following steps:

1. Copy the [sample script][sample-script] and paste it into a new file.
2. Save the file as a .ps1 file.
3. Open a PowerShell command shell.
4. Run the script with the necessary parameters to create an ExpressRoute circuit, as shown below.

	```powershell
	.\<<scriptfilename>>.ps1 -SubscriptionId <<subscription-id>> -BaseName <<prefix-for-resources>> -Location <<azure-location>> -ExpressRouteSkuTier <<sku>> -ExpressRouteSkuFamily <<family>> -ExpressRouteServiceProviderName <<your-provider>> -ExpressRoutePeeringLocation <<provider-location>> -ExpressRouteBandwidth <<bandwidth-in-mbps>>
	```

5. Contact your provider with your circuit `ServiceKey` and wait for the circuit to be provisioned.
6. Run the script with the necessary parameters to create a VNet and connect it to the ExpressRoute circuit, as shown below.

	```powershell
	.\<<scriptfilename>>.ps1 -CreateVNet $true -VnetAddressPrefix <<azure-vnet-prefix>> -InternalSubnetAddressPrefix <<azure-subnet-prefix>> -GatewaySubnetAddressPrefix <<gateway-subnet-prefix>> -SubscriptionId <<subscription-id>> -BaseName <<prefix-for-resources>> -Location <<azure-location>> -ExpressRouteSkuTier <<sku>> -ExpressRouteSkuFamily <<family>> -ExpressRouteServiceProviderName <<your-provider>> -ExpressRoutePeeringLocation <<provider-location>> -ExpressRouteBandwidth <<bandwidth-in-mbps>>
	```

## Sample solution script

The deployment steps above use the following sample script.

```powershell
param(
    [parameter(Mandatory=$true)]
    [ValidateScript({
        try {
            [System.Guid]::Parse($_) | Out-Null
            $true
        }
        catch {
            $false
        }
    })]
    [string]$SubscriptionId,

    [Parameter(Mandatory=$false)]
    [string]$BaseName = "hybrid-er",

    [Parameter(Mandatory=$false)]
    [string]$Location = "Central US",

    [Parameter(Mandatory=$false, ParameterSetName="CreateERCircuit")]
    [ValidateSet("Premium", "Standard")]
    [string]$ExpressRouteSkuTier = "Standard",

    [Parameter(Mandatory=$false, ParameterSetName="CreateERCircuit")]
    [ValidateSet("MeteredData", "UnlimitedData")]
    [string]$ExpressRouteSkuFamily = "MeteredData",

    [Parameter(Mandatory=$true, ParameterSetName="CreateERCircuit")]
    [string]$ExpressRouteServiceProviderName,

    [Parameter(Mandatory=$true, ParameterSetName="CreateERCircuit")]
    [string]$ExpressRoutePeeringLocation,

    [Parameter(Mandatory=$true, ParameterSetName="CreateERCircuit")]
    [string]$ExpressRouteBandwidth,


    [Parameter(Mandatory=$true, ParameterSetName="CreateVNet")]
    [switch]$CreateVNet,
    
    [Parameter(Mandatory=$false, ParameterSetName="CreateVNet")]
    [string]$VnetAddressPrefix = "10.20.0.0/16",

    [Parameter(Mandatory=$false, ParameterSetName="CreateVNet")]
    [string]$GatewaySubnetAddressPrefix = "10.20.255.224/27",

    [Parameter(Mandatory=$false, ParameterSetName="CreateVNet")]
    [string]$InternalSubnetAddressPrefix = "10.20.0.0/24"
)

$resourceGroup = "$BaseName-rg"
$vnetName = "$BaseName-vnet"
$internalSubnetName = "$BaseName-internal-subnet"
$expressRouteCircuitName = "$BaseName-erc"
$gatewayPublicIpAddressName = "$BaseName-pip"
$vnetGatewayName = "$BaseName-vgw"
$vpnConnectionName = "$BaseName-vpn"

Login-AzureRmAccount
Select-AzureRmSubscription -SubscriptionId $SubscriptionId

switch($PSCmdlet.ParameterSetName) {
    "CreateERCircuit" {
        New-AzureRmResourceGroup -Name $resourceGroup -Location $Location
        New-AzureRmExpressRouteCircuit -Name $expressRouteCircuitName `
            -ResourceGroupName $resourceGroup -Location $Location -SkuTier $ExpressRouteSkuTier `
            -SkuFamily $ExpressRouteSkuFamily -ServiceProviderName $ExpressRouteServiceProviderName `
            -PeeringLocation $ExpressRoutePeeringLocation -BandwidthInMbps $ExpressRouteBandwidth
    }
    "CreateVNet" {
        $gatewaySubnetConfig = New-AzureRmVirtualNetworkSubnetConfig -Name "GatewaySubnet" `
            -AddressPrefix $GatewaySubnetAddressPrefix
        $internalSubnetConfig = New-AzureRmVirtualNetworkSubnetConfig -Name $internalSubnetName `
            -AddressPrefix $InternalSubnetAddressPrefix
        $vnet = New-AzureRmVirtualNetwork -Name $vnetName -ResourceGroupName $resourceGroup `
            -Location $Location -AddressPrefix $VnetAddressPrefix `
            -Subnet $gatewaySubnetConfig, $internalSubnetConfig
		$gwsubnet = Get-AzureRmVirtualNetworkSubnetConfig -VirtualNetwork $vnet -Name GatewaySubnet
        $gatewayPublicIpAddress = New-AzureRmPublicIpAddress -Name $gatewayPublicIpAddressName -ResourceGroupName $resourceGroup `
            -Location $Location -AllocationMethod Dynamic
        $gatewayIpConfig = New-AzureRmVirtualNetworkGatewayIpConfig -Name gwIpConfig `
            -SubnetId $gwsubnet.Id -PublicIpAddressId $gatewayPublicIpAddress.Id
        $vnetGateway = New-AzureRmVirtualNetworkGateway -Name $vnetGatewayName `
            -ResourceGroupName $resourceGroup -Location $Location -IpConfigurations $gatewayIpConfig `
            -GatewayType ExpressRoute -VpnType RouteBased
        $expressRouteCircuit = Get-AzureRmExpressRouteCircuit -Name $expressRouteCircuitName `
            -ResourceGroupName $resourceGroup
        $vpnConnection = New-AzureRmVirtualNetworkGatewayConnection -Name $vpnConnectionName `
            -ResourceGroupName $resourceGroup -Location $Location -VirtualNetworkGateway1 $vnetGateway `
            -PeerId $expressRouteCircuit.Id -ConnectionType ExpressRoute
    }
}

```

<!-- links -->
[expressroute-technical-overview]: ../expressroute/expressroute-introduction.md
[circuits-and-routing-domains]: ../expressroute/expressroute-circuit-peerings.md
[resource-manager-overview]: ../resource-group-overview.md
[azure-powershell]: ../powershell-azure-resource-manager.md
[expressroute-prereqs]: ../expressroute/expressroute-prerequisites.md
[create-expressroute-circuit]: ../expressroute/expressroute-howto-circuit-arm.md
[configure-expressroute-routing]: ../expressroute/expressroute-howto-routing-arm.md
[sla-for-expressroute]: https://azure.microsoft.com/support/legal/sla/expressroute/v1_0/
[datacenter-ip-ranges]: http://www.microsoft.com/download/details.aspx?id=41653
[link-vnet-to-expressroute]: ../expressroute/expressroute-howto-linkvnet-arm.md
[ExpressRoute-provisioning]: ../expressroute/expressroute-workflows.md
[expressroute-routing-requirements]: ../expressroute/expressroute-routing.md
[expressroute-locations]: ../expressroute/expressroute-locations.md
[expressroute-pricing]: https://azure.microsoft.com/pricing/details/expressroute/
[expressroute-limits]: ../azure-subscription-service-limits/#networking-limits
[sample-script]: #sample-solution-script
[connectivity-providers]: ../expressroute/expressroute-introduction/#how-can-i-connect-my-network-to-microsoft-using-expressroute
