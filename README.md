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
    -   Windows Server 2016 VM - 10.100.0.5
    -   Azure Files Private Endpoint - 10.100.0.9 mapped to Standard
        Storage Account in Central US region.
-  Robocopy /MT (multithread) and file with 1GB of size. (**Note:** never use Explorer for file copy because it uses SMB single thread/channel) 

![](.//media/image1.jpeg)

### Networking validation (Latency)

An important aspect to take into consideration on network troubleshooting is to eliminate issues by network layers. In this case for layer 4 (transport), we can probe TCP 445 ports between client and destinations and check latency.

**Between client and remote Windows VM (10.100.0.5) over SMB (TCP 445)**

![](.//media/image2.jpeg)

**Between client and remote Azure Files Private Endpoint (10.100.0.9) 
over SMB (TCP 445)**

![](.//media/image3.jpeg)

The latency above indicates it is not a network issue that could impact on file copy between Windows and Azure File over the private endpoint. **Both have 23ms** of latency.

From a networking perspective, that would be an elimination point for a potential layer 4 issue, because we have similar latency on both connections. For that reason, the slow compare copy performance difference should be treated as application-level troubleshooting, covered in the next section.

### Application-Level validation

The first step was to dump SMB settings while connected to both Windows VM (10.100.0.5) and Private Endpoint (dmauserdemo.file.core.windows.net – 10.100.0.9). We see both using SMB3 as dialect, being the SMB version that offers the best performance.

![](.//media/image4.jpeg)

File copy simulation from On-Prem Client to:   
***Azure Files Private Endpoint (10.100.0.9)* using robopy /MT (time
taken: 57 seconds)**

 ![](.//media/image5.jpeg)

The copy process confirms that an establishment of a single channel/TCP conversation:

![](.//media/image6.jpeg)

 
File copy simulation from On-Prem Client to:  
***Windows IaaS VM (10.100.0.5)* using robocopy /mt (time taken: 33
seconds):**

![](.//media/image7.jpeg)
 

The copy process confirms that the establishment of four TCP conversations. That is an indication of the use of SMB multichannel 

![](.//media/image9.jpeg)

A simple way to verify if SMB Multichannel is active is to run the PowerShell command **Get-SmbMultichannelConnection | fl** on client-side and we have the confirmation it is active and using four channels to Windows IaaS VM (10.100.0.5), as shown:

![](.//media/image8.jpeg)

### Validation outcome
Based on the tests detailed above, there is no networking issue because
the latency on both scenarios is similar. However, when transferring
files over Windows IaaS VM uses a multichannel feature by default, and
that makes four extras TCP concurrent connections that make file
transfer much faster. 

The support for the multichannel feature on Azure
Files require Premium SKU as well as is currently in preview and limited today to North Central, South Central, and West US, see details in: [SMB Multichannel performance - Azure Files |
Microsoft
Docs](https://docs.microsoft.com/en-us/azure/storage/files/storage-files-smb-multichannel-performance)

Also, note that the Azure Files multichannel feature has limited support to Private Endpoints at this time, as shown: 

![](.//media/image10.png)

### Consideration on Storage Premium SKU 

A final consideration to keep in mind is Azure Files Storage SKU (Standard versus Premium) has also different outcomes on performance during an SMB file copy. For all validations made for this lab, we had an Azure Files storage account using Standard SKU. After we created a Premium SKU, even without a multichannel feature, it gives much better performance if compared with Standard. Here is a robocopy output of the same file transfer when a Private Endpoint was mapped to Azure Files Storage SKU:

![](.//media/image11.png)

As you see above, it brings down the total copy time of 1GB file from 57 seconds against Standard Storage to 30 seconds to Premium storage. 

## Conclusion

The main reason for the difference between Azure Files and Windows IaaS VM on file copy is that Windows uses SMB3 multichannel feature compare to Azure Files standard storage that does not have that capability. Also, it is not possible to use the multichannel feature at this time over Private Endpoints. This post illustrated the main issue was not networking related, but it was an application-level (Azure Files) current limitation. However, leveraging Premium storage SKU offers mostly double performance compared to Standard SKU.
