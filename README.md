# Traffic Flow in Common Azure Networking Patterns

## Overview
One of the of the most critical foundational decisions an organization will make when adopting Azure is settling on a networking architecture. 
The [Microsoft Azure Cloud Adoption Framework](https://docs.microsoft.com/en-us/azure/architecture/framework/security/design-network-segmentation) and [Azure Architecture Center](https://docs.microsoft.com/en-us/azure/architecture/) can help you to align your organizational requirements for security and operations with the appropriate architecture. While these resources do a great job explaining the benefits and considerations of each architecture, they often lack details as to how a packet gets from point A to point B. 

The traffic flows documented in this repository seek to fill this gap to provide the details of how the traffic typically flows and the options available to influence these flows to achieve security and operational goals. Additionally, they can act as a tool for learning the platform and troubleshooting issues with Azure networking.

This respository will be continually updated to include new flows.

## Sections
* Hub and Spoke with single NVA stack for all traffic
  * [On-premises to Azure](#single-nva-on-premises-to-azure)
  * Azure to Azure
  * Azure to Internet (Public IP)
  * Azure to Internet (NAT Gateway)
  * Internet to Azure (HTTP/HTTPS Traffic)
  * Internet to Azure (HTTP/HTTPS Traffic with NVA IDS/IPS)
  * Internet to Azure (Non-HTTP/HTTPS Traffic)
* Hub and Spoke with separate NVA stacks for east/west and north/south traffic
  * Azure to Azure
  * Azure to Internet (Public IP)
  * Azure to Internet (NAT Gateway)

## Hub and Spoke with Single NVA Stack for all traffic
The patterns in this section assume the organization is deploying a single NVA stack that will handle north/south (to and from Internet) and east/west (to and from on-premises or within Azure spoke to spoke). Each NVA is configured with a single NIC (network interface card) for payload traffic. The NVAs may have a separate NIC for management traffic, but note this NIC is not represented in these diagrams.

![HS-1NVA](https://github.com/mattfeltonma/azure-networking-patterns/blob/main/images/HS-1NVA-Image1.png)

### Single NVA On-premises to Azure
| Step | Path  | Description |
| ------------- | ------------- | ------------- |
| 1 | A -> B | Machine traffic passes over ExpressRoute circuit to Virtual Network Gateway |
| 2 | B -> G  | User defined route in route table assigned to GatewaySubnet directs traffic to internal load balancer for NVA |
| 3 | G -> F | Internal load balancer passes traffic to NVA |
| 4 | F -> H | NVA evaluates its rules, allows traffic, and passes it to internal load balancer for frontend application |
| 5 | H -> I | Internal load balancer for frontend application passes traffic to frontend application virtual machine |
| 6 | I -> G | User defined route in route table assigned to frontend subnet directs traffic to internal load balancer for NVA |
| 7 | G -> F | Internal load balancer passes traffic to NVA
| 8 | F -> B | NVA passes traffic to Virtual Network Gateway 
| 9 | B -> A | Virtual Network Gateway passes traffic over ExpressRoute circuit back to machine on-premises |
