**Issue:**

Copying files from On-Prem via S2S IPSec tunnel is slower over Private
Endpoint compared to Windows IaaS VM even when both are on Azure Central
US VNET. Based on the tests below, there is no networking issue because
the latency on both scenarios is similar. However, when transferring
files over Windows IaaS VM uses a multichannel feature by default, and
that makes four extras TCP concurrent connections that make file
transfer much faster. The multichannel feature is supported on Azure
Files Premium but is limited today to North Central, South Central, and
West US, see details in: [SMB Multichannel performance - Azure Files |
Microsoft
Docs](https://docs.microsoft.com/en-us/azure/storage/files/storage-files-smb-multichannel-performance)

As side note, other important networking consideration for Azure Files
are documented here:

[Azure Files networking considerations | Microsoft
Docs](https://docs.microsoft.com/en-us/azure/storage/files/storage-files-networking-overview)

 

 

**Technical Validation**

 

**Scenario:**

-   OnPrem Network (10.20.2.0/24)

    -   On-Prem VM Client (Windows) - 10.20.2.6

-   S2S IPSec VPN between On-Prem and Azure.

-   Azure VNET (10.100.0.0/24) - on Central US region

    -   Windows Server VM - 10.100.0.5

    -   Azure Files Private Endpoint - 10.100.0.9 mapped to Standard
        Storage Account in Central US

-   Used robocopy /MT (multithread) and file with 1GB size.

 

![](.//media/image1.jpeg){width="6.5in" height="2.1590277777777778in"}

**Latency Validation:**

 

Between client and remote Windows VM (10.100.0.5) over SMB (TCP 445)

 

![](.//media/image2.jpeg){width="5.491666666666666in"
height="2.091666666666667in"}

 

Between client and remote Azure Files Private Endpoint (10.100.0.9) 
over SMB (TCP 445)

![](.//media/image3.jpeg){width="5.558333333333334in" height="2.225in"}

 

 

Latency above indicates there's not really a network issue that could
impact on file copy between Windows and Azure File over private
endpoint. **Both have 23ms** of latency.

From a networking perspective that would the point of elimination for
layer 4 issue because we have similar latency on both connections.

 

**Application-Level testing**

 

Dumped SMB settings while connected to both Windows VM (10.100.0.5) and
Private Endpoint (dmauserdemo.file.core.windows.net – 10.100.0.9)

![](.//media/image4.jpeg){width="4.541666666666667in" height="2.325in"}

 

***Azure Files Private Endpoint (10.100.0.9)* using robopy /MT (time
taken: 57 seconds)**

 

![](.//media/image5.jpeg){width="5.85in" height="1.8in"}

 

Confirmed single channel/TCP conversation is being used:

![](.//media/image6.jpeg){width="6.5in" height="0.8055555555555556in"}

 

***Windows IaaS VM (10.100.0.5)* using robocopy /mt (time taken: 33
seconds):**

![](.//media/image7.jpeg){width="6.041666666666667in"
height="1.8333333333333333in"}

 

Confirmed for Windows IaaS VM (10.100.0.5) SMB Multichannel is being
used.

![](.//media/image8.jpeg){width="4.491666666666666in"
height="2.908333333333333in"}

 

This show multi channel is being used because we have four threads:

![](.//media/image9.jpeg){width="6.25in" height="1.2916666666666667in"}

 

 

**Conclusion**

The main reason for the difference between Azure Files and Windows IaaS
VM on file copy is Windows uses SMB3 multichannel feature whereas Azure
Files on Central US does not have that capability. Therefore, that
behavior is expected by design behavior and further details and solution
on how Azure Files can leverage SMB3 multichannel feature documented on
[**SMB Multichannel performance - Azure Files | Microsoft
Docs**](https://docs.microsoft.com/en-us/azure/storage/files/storage-files-smb-multichannel-performance)
