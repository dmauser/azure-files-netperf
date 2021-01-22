# Network performance considerations when using Azure Files over Private Endpoint

## Introduction

The goal of this post is to describe file copy performance validation between On-premises and Azure Files, but before we go on the more specifics, I suggest that you get familiar with some specific Azure Files networking concepts covered in: [Azure Files networking considerations | Microsoft
Docs](https://docs.microsoft.com/en-us/azure/storage/files/storage-files-networking-overview)

In short, remember Azure Files uses SMB (TCP 445) for file copy and HTTPS (TCP 443) for file replication. In our case here SMB is going to be the target protocol for performance validation.

## Background

In this case scenario, the customer reports that when copying files from On-Prem via S2S IPSec tunnel is slower over Private Endpoint compared when copying the same file (1 GB) over a Windows IaaS VM. Both Private Endpoint and VM are on the same Azure Virtual Network (VNET) in US Central US region.

## Technical Validation

**Scenario:**
-   OnPrem Network (10.20.2.0/24)
    -   On-Prem VM Client (Windows) - 10.20.2.6
-   S2S IPSec VPN between On-Prem and Azure.
-   Azure VNET (10.100.0.0/24) - on Central US region
    -   Windows Server VM - 10.100.0.5
    -   Azure Files Private Endpoint - 10.100.0.9 mapped to Standard
        Storage Account in Central US region.
-   Used robocopy /MT (multithread) and file with 1GB size. (**Note:** never use Explorer for file copy because it uses SMB single thread/channel) 

![](.//media/image1.jpeg)

### Networking validation (Latancy)

Very important aspect to take in consideration on network troubleshooting is to eliminate issues by network layers. In this case for layer 4 (transport) we can probe TCP 445 ports between client and destinations and check latency.

**Between client and remote Windows VM (10.100.0.5) over SMB (TCP 445)**

![](.//media/image2.jpeg)

**Between client and remote Azure Files Private Endpoint (10.100.0.9) 
over SMB (TCP 445)**

![](.//media/image3.jpeg)

Latency above indicates there's not really a network issue that could
impact on file copy between Windows and Azure File over private
endpoint. **Both have 23ms** of latency.

From a networking perspective that would the point of elimination for potential layer 4 issues, because we have similar latency on both connections. For that reason now issue should be treated as application-level troubleshooting, covered on the next section.

### Application-Level validation

First step was to dump SMB settings while connected to both Windows VM (10.100.0.5) and Private Endpoint (dmauserdemo.file.core.windows.net – 10.100.0.9). We see both using SMB3 as dialect which is the SMB version that offers best performance.

![](.//media/image4.jpeg)


File copy simulation from On-Prem Client to:   
***Azure Files Private Endpoint (10.100.0.9)* using robopy /MT (time
taken: 57 seconds)**

 ![](.//media/image5.jpeg)

It has been confirmed a single channel/TCP conversation is being used:

![](.//media/image6.jpeg)

 
File copy simulation from On-Prem Client to:  
***Windows IaaS VM (10.100.0.5)* using robocopy /mt (time taken: 33
seconds):**

![](.//media/image7.jpeg)
 

Confirmed for Windows IaaS VM (10.100.0.5) SMB Multichannel is being
used.

![](.//media/image8.jpeg)

This show multi channel is being used because we have four TCP connections (four threads):

![](.//media/image9.jpeg)

### Validation outcome
Based on the tests detailed below, there is no networking issue because
the latency on both scenarios are similar. However, when transferring
files over Windows IaaS VM uses a multichannel feature by default, and
that makes four extras TCP concurrent connections that make file
transfer much faster. 

The multichannel feature is supported on Azure
Files Premium but is limited today to North Central, South Central, and
West US, see details in: [SMB Multichannel performance - Azure Files |
Microsoft
Docs](https://docs.microsoft.com/en-us/azure/storage/files/storage-files-smb-multichannel-performance)

Also it is important to not that Azure Files multi-channel feature is currently not supported over Private Endpoints, limitation that is included on the link above. 

![](.//media/image10.png)

### Consideration on Storage Premium SKU 

A final consideration to keep in mind is Azure Files Storage SKU (Standard verus Premium) has also different outcomes on performance during a SMB file copy. All validation done for this lab we had a Azure Files storage account using Standard SKU. After we created a Premium SKU, even without multichannel feature, it gives much better performance if compared with Standard. Here is a robocopy output of the same file transfer when a Private Endpoint was mapped to Azure Files Storage SKU:

![](.//media/image11.png)

As you see it brings down total copy time of 1GB file from 57 seconds against Standard Storage to 30 seconds to Premium storage. 

## Conclusion

The main reason for the difference between Azure Files and Windows IaaS
VM on file copy is Windows uses SMB3 multichannel feature whereas Azure
Files on Central US does not have that capability as well as is not possible to use at this time over Private Endpoints. This example illustrate that is not a networking issue by  application level (Azure Files) current limitation. However, leveraging Premium storage SKU offers mostly double performance compared to Standard SKU.
