# Traffic Flow in Common Azure Networking Patterns

## Overview
One of the of the most critical foundational decisions an organization will make when adopting Azure is settling on a networking architecture. 
The [Microsoft Azure Cloud Adoption Framework](https://docs.microsoft.com/en-us/azure/architecture/framework/security/design-network-segmentation) and [Azure Architecture Center](https://docs.microsoft.com/en-us/azure/architecture/) can help you to align your organizational requirements for security and operations with the appropriate architecture. While these resources do a great job explaining the benefits and considerations of each architecture, they often lack details as to how a packet gets from point A to point B. 

The traffic flows documented in this repository seek to fill this gap to provide the details of how the traffic typically flows and the options available to influence these flows to achieve security and operational goals. Additionally, they can act as a tool for learning the platform and troubleshooting issues with Azure networking.

This respository will be continually updated to include new flows.

## Sections
* [Hub and Spoke with single NVA stack for all traffic](#hub-and-spoke-with-single-nva-stack-for-all-traffic)
  * [On-premises to Azure](#single-nva-on-premises-to-azure)
  * [Azure to Azure](#single-nva-azure-to-azure)
  * [Azure to Internet using Public IP](#single-nva-azure-to-internet-using-public-ip)
  * [Azure to Internet using NAT Gateway](#single-nva-azure-to-internet-using-nat-gateway)
  * [Internet to Azure with HTTP/HTTPS Traffic](#single-nva-internet-to-azure-http-and-https)
  * [Internet to Azure with HTTP/HTTPS Traffic with IDS IPS](#single-nva-internet-to-azure-http-and-https-with-ids-ips)
  * [Internet to Azure Non HTTP/HTTPS Traffic](#single-nva-internet-to-azure-non-http-and-https)
* Hub and Spoke with separate NVA stacks for east/west and north/south traffic
  * Azure to Azure
  * Azure to Internet (Public IP)
  * Azure to Internet (NAT Gateway)
* Hub and Spoke with single NVA stack for all traffic and NVA has dual NICs
  * On-premises to Azure
* Hub and Spoke with single NVA stack for all traffic in multiple regions
  * Azure to Azure

## Hub and Spoke with Single NVA Stack for all traffic
The patterns in this section assume the organization is deploying a single NVA stack that will handle north/south (to and from Internet) and east/west (to and from on-premises or within Azure spoke to spoke). Each NVA is configured with a single NIC (network interface card) for payload traffic. The NVAs may have a separate NIC for management traffic, but note this NIC is not represented in these diagrams.

### Single NVA On-premises to Azure
Scenario: Machine on-premises initiates a connection to an application running in Azure.
![HS-1NVA](https://github.com/mattfeltonma/azure-networking-patterns/blob/main/images/HS-1NVA-Image1.svg)

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

### Single NVA Azure to Azure
Scenario: Virtual machine in one spoke initiates connection to virtual machine in another spoke.
![HS-1NVA](https://github.com/mattfeltonma/azure-networking-patterns/blob/main/images/HS-1NVA-Image1.svg)

| Step | Path  | Description |
| ------------- | ------------- | ------------- |
| 1 | I -> G | User defined route in route table assigned to frontend subnet directs traffic to internal load balancer for NVA |
| 2 | G -> F | Internal load balancer passes traffic to NVA |
| 3 | F -> L | NVA evaluates its rules, allows traffic, and passes it to Active Directory domain controller virtual machine |
| 4 | L -> G | User defined route in route table assigned to snet-ad subnet directs traffic to internal load balancer for NVA |
| 5 | G -> F | Internal load balancer passes traffic to NVA |
| 6 | F -> I | NVA passes traffic back to frontend virtual machine |

### Single NVA Azure to Internet using Public IP
Scenario: Virtual machine in Azure initiates a connection to a third-party website on the Internet and the NVA is configured with public IPs.
![HS-1NVA](https://github.com/mattfeltonma/azure-networking-patterns/blob/main/images/HS-1NVA-Image1.svg)

| Step | Path  | Description |
| ------------- | ------------- | ------------- |
| 1 | I -> G | User defined route in route table assigned to frontend subnet directs traffic to internal load balancer for NVA |
| 2 | G -> F | Internal load balancer passes traffic to NVA |
| 3 | F -> @ | NVA evaluates its rules, allows traffic, NATs to its public IP, and passes traffic to third-party website |
| 4 | @ -> D | Third-party website passes traffic back to public IP of NVA |
| 5 | F -> I | NVA passes traffic to frontend virtual machine |

### Single NVA Azure to Internet using NAT Gateway
Scenario: Virtual machine in Azure initiates a connection to a third-party website on the Internet and the NVAs are configured to use NAT Gateway.
![HS-1NVA](https://github.com/mattfeltonma/azure-networking-patterns/blob/main/images/HS-1NVA-Image1-NG.svg)

| Step | Path  | Description |
| ------------- | ------------- | ------------- |
| 1 | I -> G | User defined route in route table assigned to frontend subnet directs traffic to internal load balancer for NVA |
| 2 | G -> F | Internal load balancer passes traffic to NVA |
| 3 | F -> D | NVA evaluates its rules, allows traffic, and passes traffic to NAT Gateway |
| 4 | E -> @ | NAT Gateway NATs to its public IP and passes traffic to third-party website |
| 5 | @ -> E | Third-party website passes traffic back to public IP of NAT Gateway |
| 6 | D -> F | NAT Gateway passes traffic to NVA |
| 7 | F -> I | NVA passes traffic to frontend virtual machine |

### Single NVA Internet to Azure Http and Https
Scenario: User on the Internet initiates a connection to an application running in Azure. The application has been secured behind an Application Gateway for intra-region security and load balancing. Azure Front Door is placed in front of the Application Gateway to provide inter-region security, load balancing, and site acceleration.
![HS-1NVA](https://github.com/mattfeltonma/azure-networking-patterns/blob/main/images/HS-1NVA-Web-Inbound-No-Ids-Ips.svg)
| Step | Path  | Description |
| ------------- | ------------- | ------------- |
| 1 | @ -> P | User's machine sends traffic to Azure Front Door which terminates the TCP connection |
| 2 | P -> O | Azure Front Door establishes a new TCP connection with the Application Gateway's public IP and adds the user's public IP to the X-Forwarded-For header |
| 3 | N -> H | Application Gateway NATs to its private IP, appends the X-Forwarded-Header with the Azure Front Door public IP, performs its security and load balancing function, and passes the traffic to the web frontend internal load balancer |
| 4 | H -> I | Internal load balancer passes traffic to the web frontend virtual machine |
| 5 | I -> N | Web frontend virtual machine passes traffic to the Application Gateway private IP |
| 6 | O -> P | Application Gateway NATs to its public IP and passes traffic to Azure Front Door |
| 7 | P -> @ | Azure Front Door passes traffic to user's machine |

### Single NVA Internet to Azure Http and Https with IDS IPS
Scenario: User on the Internet initiates a connection to an application running in Azure. The application has been secured behind an Application Gateway for intra-region security and load balancing. Azure Front Door is placed in front of the Application Gateway to provide inter-region security, load balancing, and site acceleration. An NVA is placed between the Application Gateway and the application to provide IDS/IPS functionality. 

Reference the [public documentation](https://docs.microsoft.com/en-us/azure/architecture/example-scenario/gateway/firewall-application-gateway) for additional ways to achieve this pattern.
![HS-1NVA](https://github.com/mattfeltonma/azure-networking-patterns/blob/main/images/HS-1NVA-Web-Inbound-Ids-Ips.svg)
| Step | Path  | Description |
| ------------- | ------------- | ------------- |
| 1 | @ -> P | User's machine sends traffic to Azure Front Door which terminates the TCP connection |
| 2 | P -> O | Azure Front Door establishes a new TCP connection with the Application Gateway's public IP and adds the user's public IP to the X-Forwarded-For header |
| 3 | N -> G | Application Gateway NATs to its private IP, appends the X-Forwarded-Header with the Azure Front Door public IP, performs its security and load balancing function, and user defined route in route table assigned to Application Gateway subnet directs traffic to the internal load balancer for the NVA |
| 4 | G -> F | Internal load balancer passes traffic to NVA |
| 5 | F -> H | NVA evaluates its rules, allows traffic, and passes it to the web frontend internal load balancer |
| 6 | H -> I | Internal load balancer passes traffic to the frontend virtual machine |
| 7 | I -> G | User defined route in route table assigned to web frontend subnet directs traffic to the internal load balancer for the NVA |
| 8 | G -> F | Internal load balancer passes traffic to the NVA |
| 9 | F -> N | NVA passes traffic to the Application Gateway private IP |
| 10 | O -> P | Application Gateway NATs to its public IP and passes traffic to Azure Front Door |
| 11 | P -> @ | Azure Front Door passes traffic to user's machine |

### Single NVA Internet to Azure Non Http and Https
Scenario: User on the Internet initiates a connection to an application running in Azure. The application is served up using a protocol that IS NOT HTTP/HTTPS. An NVA is placed between the Internet and the application.
![HS-1NVA](https://github.com/mattfeltonma/azure-networking-patterns/blob/main/images/HS-1NVA-Non-Web-Inbound.svg)
| Step | Path  | Description |
| ------------- | ------------- | ------------- |
| 1 | @ -> C | User's machine sends traffic to the public IP address of an external Azure Load Balancer |
| 2 | C -> F | External load balancer passes the traffic through the Azure software-defined-network to the NVA |
| 3 | F -> H | NVA evaluates its rules, allows traffic, NATs to its private IP, and passes it to the web frontend internal load balancer |
| 4 | H -> I | Web frontend internal load balancer passes traffic to web frontend virtual machine |
| 5 | I -> F | Web frontend virtual machine passes traffic to NVA |
| 6 | F -> @ | NVA passes traffic back through the Azure software-defined-network where its source IP address is rewritten to the external load balancer's public IP
