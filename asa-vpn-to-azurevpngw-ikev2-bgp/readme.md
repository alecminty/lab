# Azure Networking Lab- IPSEC VPN (IKEv2) between Cisco ASAv and Azure VPN Gateway with BGP

This lab guide illustrates how to build a basic IPSEC VPN tunnel w/IKEv2 between a Cisco ASAv and the Azure VPN gateway with BGP. This is for lab testing purposes only. NAT and security proposals shown can be narrowed down if need be. All Azure configs are done in Azure CLI so you can change them as needed to match your environment. Note- the on prem ASAv has a private IP on the outside interface since it's hosted in Azure. You can apply a public IP if needed. The on prem VNET is to simulate on prem connectivity.

Assumptions:
- A valid Azure subscription account. If you don’t have one, you can create your free azure account (https://azure.microsoft.com/en-us/free/) today.
- Latest Azure CLI, follow these instructions to install: https://docs.microsoft.com/en-us/cli/azure/install-azure-cli 


# Base Topology
The lab deploys an Azure VPN gateway into a VNET. We will also deploy a Cisco CSR in a seperate VNET to simulate on prem.
![alt text](https://github.com/jwrightazure/lab/blob/master/images/asavlab.png)
 

**Build Resource Groups, VNETs and Subnets**
<pre lang="...">
az group create --name Hub --location eastus
az network vnet create --resource-group Hub --name Hub --location eastus --address-prefixes 10.0.0.0/16 --subnet-name HubVM --subnet-prefix 10.0.10.0/24
az network vnet subnet create --address-prefix 10.0.0.0/24 --name GatewaySubnet --resource-group Hub --vnet-name Hub
</pre>

**Build Resource Groups, VNETs and Subnets to simulate on prem**
<pre lang="...">
az group create --name onprem --location eastus
az network vnet create --resource-group onprem --name onprem --location eastus --address-prefixes 10.1.0.0/16 --subnet-name VM --subnet-prefix 10.1.10.0/24
az network vnet subnet create --address-prefix 10.1.0.0/24 --name zeronet --resource-group onprem --vnet-name onprem
az network vnet subnet create --address-prefix 10.1.1.0/24 --name onenet --resource-group onprem --vnet-name onprem
az network vnet subnet create --address-prefix 10.1.2.0/24 --name twonet --resource-group onprem --vnet-name onprem
</pre>

**Build Azure side Linux VM**
<pre lang="...">
az network public-ip create --name HubVMPubIP --resource-group Hub --location eastus --allocation-method Dynamic
az network nic create --resource-group Hub -n HubVMNIC --location eastus --subnet HubVM --private-ip-address 10.0.10.10 --vnet-name Hub --public-ip-address HubVMPubIP
az vm create -n HubVM -g Hub --image UbuntuLTS --admin-username azureuser --admin-password Msft123Msft123 --nics HubVMNIC
</pre>

**Build onprem side Linux VM**
<pre lang="...">
az network public-ip create --name onpremVMPubIP --resource-group onprem --location eastus --allocation-method Dynamic
az network nic create --resource-group onprem -n onpremVMNIC --location eastus --subnet VM --private-ip-address 10.1.10.10 --vnet-name onprem --public-ip-address onpremVMPubIP
az vm create -n onpremVM -g onprem --image UbuntuLTS --admin-username azureuser --admin-password Msft123Msft123 --nics onpremVMNIC
</pre>

**Build Public IPs for Azure VPN Gateway**
<pre lang="...">
az network public-ip create --name Azure-VNGpubip --resource-group Hub --allocation-method Dynamic
</pre>

**Build Azure VPN Gateway. Deployment will take some time. Azure side BGP ASN is 65001**
<pre lang="...">
az network vnet-gateway create --name Azure-VNG --public-ip-address Azure-VNGpubip --resource-group Hub --vnet Hub --gateway-type Vpn --vpn-type RouteBased --sku VpnGw1 --no-wait --asn 65001
</pre>

**Before deploying ASA in the next step, you may have to accept license agreement unless you have used it before. You can accomplish this through deploying a ASA in the portal or Powershell commands**
<pre lang="...">
Sample Powershell for CSR:
Get-AzureRmMarketplaceTerms -Publisher "Cisco" -Product "cisco-csr-1000v" -Name "16_10-byol"
Get-AzureRmMarketplaceTerms -Publisher "Cisco" -Product "cisco-csr-1000v" -Name "16_10-byol" | Set-AzureRmMarketplaceTerms -Accept
</pre>


**Build on prem ASAv. ASAv image is specified from the Marketplace in this example.**
<pre lang="...">
az network public-ip create --name ASA1MgmtIP --resource-group onprem --idle-timeout 30 --allocation-method Static
az network public-ip create --name ASA1VPNPublicIP --resource-group onprem --idle-timeout 30 --allocation-method Static
az network nic create --name ASA1MgmtInterface -g onprem --subnet twonet --vnet onprem --public-ip-address ASA1MgmtIP --private-ip-address 10.1.2.4 --ip-forwarding true
az network nic create --name ASA1OutsideInterface -g onprem --subnet zeronet --vnet onprem --public-ip-address ASA1VPNPublicIP --private-ip-address 10.1.0.4 --ip-forwarding true
az network nic create --name ASA1InsideInterface -g onprem --subnet onenet --vnet onprem --private-ip-address 10.1.1.4 --ip-forwarding true
az vm create --resource-group onprem --location eastus --name ASA1 --size Standard_D3_v2 --nics ASA1MgmtInterface ASA1OutsideInterface ASA1InsideInterface  --image cisco:cisco-asav:asav-azure-byol:910.1.0 --admin-username azureuser --admin-password Msft123Msft123
</pre>

**After the gateway and ASAv have been created, document the public IP address for both. Value will be null until it has been successfully provisioned. Please note that the ASA VPN interface and management interfaces are different**
<pre lang="...">
az network public-ip show -g Hub -n Azure-VNGpubip --query "{address: ipAddress}"
az network public-ip show -g onprem -n ASA1VPNPublicIP --query "{address: ipAddress}"
az network public-ip show -g onprem -n ASA1MgmtIP --query "{address: ipAddress}"
</pre>

**Verify BGP information on the Azure VPN GW**
<pre lang="...">
az network vnet-gateway list --query [].[name,bgpSettings.asn,bgpSettings.bgpPeeringAddress] -o table --resource-group Hub
</pre>

**Create a route table and routes for the Azure VNET with correct association. This is for the onprem simulation to route traffic to the ASAv.**
<pre lang="...">
az network route-table create --name vm-rt --resource-group onprem
az network route-table route create --name vm-rt --resource-group onprem --route-table-name vm-rt --address-prefix 10.0.0.0/16 --next-hop-type VirtualAppliance --next-hop-ip-address 10.1.1.4
az network vnet subnet update --name VM --vnet-name onprem --resource-group onprem --route-table vm-rt
</pre>

**Create Local Network Gateway. On prem BGP peer over IPSEC is in ASN 65002.**
<pre lang="...">
az network local-gateway create --gateway-ip-address "insert ASA Public IP" --name to-onprem --resource-group Hub --local-address-prefixes 192.168.2.1/32 --asn 65002 --bgp-peering-address 192.168.2.1
</pre>

**Create VPN connections**
<pre lang="...">
az network vpn-connection create --name to-onprem --resource-group Hub --vnet-gateway1 Azure-VNG -l eastus --shared-key Msft123Msft123 --local-gateway2 to-onprem --enable-bgp
</pre>

**SSH to ASAv public IP. Change any reference of the Azure VPN gateway to the public IP address from previous commands.**
<pre lang="...">
interface GigabitEthernet0/0
 nameif outside
 security-level 0
 ip address 10.1.0.4 255.255.255.0 
 no shut
!
interface GigabitEthernet0/1
 nameif inside
 security-level 100
 ip address 10.1.1.4 255.255.255.0
 no shut

!By default, ASAv has default route (burned in) pointing out the Mgmt interface. Route Azure VPN GW out the outside interface which we're using for VPN termination
route OUTSIDE ""Azure-VNGpubip"" 255.255.255.255 10.1.0.1 1

!route traffic from the ASAv destin for the on prem subnet to the fabric
route inside 10.1.10.0 255.255.255.0 10.1.1.1 1

!Must create crypto profiles first
crypto ipsec ikev2 ipsec-proposal Azure-Ipsec-PROP-to-onprem
 protocol esp encryption aes-256 aes-192 aes
 protocol esp integrity sha-256 sha-1
crypto ipsec ikev2 ipsec-proposal AES256
 protocol esp encryption aes-256
 protocol esp integrity sha-1 md5
crypto ipsec ikev2 ipsec-proposal AES192
 protocol esp encryption aes-192
 protocol esp integrity sha-1 md5
crypto ipsec ikev2 ipsec-proposal AES
 protocol esp encryption aes
 protocol esp integrity sha-1 md5
crypto ipsec ikev2 ipsec-proposal 3DES
 protocol esp encryption 3des
 protocol esp integrity sha-1 md5
crypto ipsec ikev2 ipsec-proposal DES
 protocol esp encryption des
 protocol esp integrity sha-1 md5
crypto ipsec profile Azure-Ipsec-PROF-to-onprem
 set ikev2 ipsec-proposal Azure-Ipsec-PROP-to-onprem
 set security-association lifetime kilobytes unlimited
 set security-association lifetime seconds 3600
crypto ipsec security-association lifetime seconds 3600
crypto ipsec security-association lifetime kilobytes unlimited
crypto ipsec security-association replay disable
crypto ipsec security-association pmtu-aging infinite
crypto ca trustpool policy
crypto isakmp disconnect-notify

crypto ikev2 policy 1
 encryption aes-256 aes-192 aes 3des
 integrity sha256 sha
 group 2
 prf sha256 sha
 lifetime seconds 28800

!Tunnel IP must not conflict with any prefixes. This can be a /30
interface Tunnel11
 nameif vti-to-onprem
 ip address 192.168.2.1 255.255.255.0 
 tunnel source interface OUTSIDE
 tunnel destination ""Azure-VNGpubip""
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile Azure-Ipsec-PROF-to-onprem
 no shut


!route traffic for Azure over the tunnel to the tunnel interface
route vti-to-onprem 10.0.0.254 255.255.255.255 "Azure-VNGpubip" 1

crypto ikev2 enable OUTSIDE
crypto ikev2 notify invalid-selectors

group-policy AzureGroupPolicy internal
group-policy AzureGroupPolicy attributes
 vpn-tunnel-protocol ikev2 l2tp-ipsec 
dynamic-access-policy-record DfltAccessPolicy
tunnel-group "Azure-VNGpubip" type ipsec-l2l
tunnel-group "Azure-VNGpubip" general-attributes
 default-group-policy AzureGroupPolicy
tunnel-group "Azure-VNGpubip" ipsec-attributes
 ikev2 remote-authentication pre-shared-key Msft123Msft123
 ikev2 local-authentication pre-shared-key Msft123Msft123
no tunnel-group-map enable peer-ip
tunnel-group-map default-group "Azure-VNGpubip"
!
class-map inspection_default
 match default-inspection-traffic

router bgp 65002
 bgp log-neighbor-changes
 bgp graceful-restart
 bgp router-id 192.168.2.1
 address-family ipv4 unicast
  neighbor 10.0.0.254 remote-as 65001
  neighbor 10.0.0.254 ebgp-multihop 255
  neighbor 10.0.0.254 activate
  network 192.168.2.1 mask 255.255.255.255
  network 10.1.10.0 mask 255.255.255.0
  no auto-summary
  no synchronization
 exit-address-family
</pre>

**Generate interesting traffic to initiate tunnel**</br>
Connect to onprem VM and ping the VM in the Azure Hub VNET (10.0.10.10)

**Validate VPN connection status in Azure CLI**
<pre lang="...">
az network vpn-connection show --name to-onprem --resource-group Hub --query "{status: connectionStatus}"
</pre>

**Validate BGP routes being advetised from the Azure VPN GW to the ASA**
<pre lang="...">
az network vnet-gateway list-advertised-routes -g Hub -n Azure-VNG --peer 192.168.2.1
</pre>

**Validate BGP routes the Azure VPN GW is receiving from the ASA**
<pre lang="...">
az network vnet-gateway list-learned-routes -g Hub -n Azure-VNG
</pre>

**Manually add a new address space 1.1.1.0/24 to the Hub VNET. Create subnet 1.1.1.0/24. Make sure to name the subnet "test1".**
- Use Azure portal

**Create VM in new 1.1.1.0/24 network.**
<pre lang="...">
az network public-ip create --name test1VMPubIP --resource-group Hub --location eastus --allocation-method Dynamic
az network nic create --resource-group Hub -n test1VMNIC --location eastus --subnet test1 --private-ip-address 1.1.1.10 --vnet-name Hub --public-ip-address test1VMPubIP
az vm create -n test1VM -g Hub --image UbuntuLTS --admin-username azureuser --admin-password Msft123Msft123 --nics test1VMNIC
</pre>

- The new address space created on the existing VNET will automatically be advertised to the ASA via BGP. Once the VM is created, ping 1.1.1.10 sourcing from the On Prem VM 10.1.10.10. 

**You can also add static routes or network statements to the ASA to validate new prefixes are added to the Azure effective route table.**
<pre lang="...">
ASA1(config)# route null0 2.2.2.2 255.255.255.255
ASA1(config)# route null0 3.3.3.3 255.255.255.255
ASA1(config-router)# address-family ipv4 unicast 
ASA1(config)# router bgp 65002
ASA1(config-router-af)# network 2.2.2.2 mask 255.255.255.255
ASA1(config-router-af)# network 3.3.3.3 mask 255.255.255.255
</pre>







