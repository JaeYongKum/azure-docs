﻿---
title: Replicate and fail over VMware VMs to Azure using PowerShell in Azure Site Recovery | Microsoft Docs
description: Learn how to set up replication and failover to Azure for VMware VMs using PowerShell in Azure Site Recovery.
services: site-recovery
author: bsiva
manager: abhemraj
editor: raynew
ms.service: site-recovery
ms.topic: article
ms.date: 03/05/2018
ms.author: bsiva


---
# Replicate and fail over VMware VMs to Azure with PowerShell

In this article, you see how to replicate and failover VMware virtual machines to Azure using Azure PowerShell. 

You learn how to:

> [!div class="checklist"]
> - Create a Recovery Services vault.
> - Set the vault context.
> - Validate that your Configuration Server and Scale out Process servers are registered to the vault.
> - Create a replication policy and map it for use with the Configuration Server.
> - Add a vCenter server and discover VMware virtual machines.
> - Create storage accounts to replicate virtual machines to.
> - Replicate VMware virtual machines to Azure storage accounts.
> - Configure failover settings for replicating virtual machines.
> - Perform a test failover, validate, and cleanup test failover.
> - Failover to Azure.

## Prerequisites

Before you start:
- Make sure that you understand the [scenario architecture and components](vmware-azure-architecture.md).
- Review the [support requirements](site-recovery-support-matrix-to-azure.md) for all components.
- You have version 5.0.1 or greater of the AzureRm PowerShell module. If you need to install or upgrade Azure PowerShell, follow this [Guide to install and configure Azure PowerShell](/powershell/azureps-cmdlets-docs).

## Log in to your Microsoft Azure subscription

Log in to your Azure subscription using the Connect-AzureRmAccount cmdlet

```azurepowershell
Connect-AzureRmAccount
```
Select the Azure subscription you want to replicate your VMware virtual machines to. Use the Get-AzureRmSubscription cmdlet to get the list of Azure subscriptions you have access to. Select the Azure subscription to work with using the Select-AzureRmSubscription cmdlet.

```azurepowershell
Select-AzureRmSubscription -SubscriptionName "ASR Test Subscription"
```
## Create a Recovery Services vault

* Create a resource group in which to create the Recovery Services vault. In the example below, the resource group is named VMwareDRtoAzurePS and is created in the East Asia region.

```azurepowershell
New-AzureRmResourceGroup -Name "VMwareDRtoAzurePS" -Location "East Asia"
```
```
ResourceGroupName : VMwareDRtoAzurePS
Location          : eastasia
ProvisioningState : Succeeded
Tags              :
ResourceId        : /subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/VMwareDRtoAzurePS
```
   
* Create a Recovery services vault. In the example below, the Recovery services vault is named VMwareDRToAzurePs, and is created in the East Asia region and in the resource group created in the previous step.

```azurepowershell
New-AzureRmRecoveryServicesVault -Name "VMwareDRToAzurePs" -Location "East Asia" -ResourceGroupName "VMwareDRToAzurePs"
```
```
Name              : VMwareDRToAzurePs
ID                : /subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/VMwareDRToAzurePs/providers/Microsoft.RecoveryServices/vaults/VMwareDRToAzurePs
Type              : Microsoft.RecoveryServices/vaults
Location          : eastasia
ResourceGroupName : VMwareDRToAzurePs
SubscriptionId    : xxxxxxxx-xxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
Properties        : Microsoft.Azure.Commands.RecoveryServices.ARSVaultProperties
``` 

* Download the vault registration key for the vault. The vault registration key is used to register the on-premises Configuration Server to this vault. Registration is part of the Configuration Server software installation process.

```azurepowershell
#Get the vault object by name and resource group and save it to the $vault PowerShell variable 
$vault = Get-AzureRmRecoveryServicesVault -Name "VMwareDRToAzurePS" -ResourceGroupName "VMwareDRToAzurePS"

#Download vault registration key to the path C:\Work
Get-AzureRmRecoveryServicesVaultSettingsFile -SiteRecovery -Vault $Vault -Path "C:\Work\"
```
```
FilePath
--------
C:\Work\VMwareDRToAzurePs_2017-11-23T19-52-34.VaultCredentials
```

Use the downloaded vault registration key and follow the steps in the articles given below to complete installation and registration of the Configuration Server.
* [Choose your protection goals](vmware-azure-set-up-source.md#choose-your-protection-goals)
* [Set up the source environment](vmware-azure-set-up-source.md#set-up-the-configuration-server) 

## Set the vault context

> [!TIP]
> The Azure Site Recovery PowerShell module (AzureRm.RecoveryServices.SiteRecovery module) comes with easy to use aliases for most cmdlets. The cmdlets in the module take the form *\<Operation>-**AzureRmRecoveryServicesAsr**\<Object>* and have equivalent aliases that take the form *\<Operation>-**ASR**\<Object>*. This article uses the cmdlet aliases for ease of reading.

Set the vault context using the Set-ASRVaultContext cmdlet. Once set, subsequent Azure Site Recovery operations in the PowerShell session are performed in the context of the selected vault. In the example below, the vault details from the $vault variable is used to specify the vault context for the PowerShell session.
 ```azurepowershell
Set-ASRVaultContext -Vault $vault
```
```
ResourceName      ResourceGroupName ResourceNamespace          ResouceType
------------      ----------------- -----------------          -----------
VMwareDRToAzurePs VMwareDRToAzurePs Microsoft.RecoveryServices vaults
```

> [!TIP]
> As an alternative to the Set-ASRVaultContext cmdlet, one can also use the Import-AzureRmRecoveryServicesAsrVaultSettingsFile cmdlet to set the vault context. Specify the path at which the vault registration key file is located as the -path parameter to the Import-AzureRmRecoveryServicesAsrVaultSettingsFile cmdlet.
>
>For example:
>
>```azurepowershell
>Get-AzureRmRecoveryServicesVaultSettingsFile -SiteRecovery -Vault $Vault -Path "C:\Work\"
>
>Import-AzureRmRecoveryServicesAsrVaultSettingsFile -Path "C:\Work\VMwareDRToAzurePs_2017-11-23T19-52-34.VaultCredentials"
>```
>

Subsequent sections of this article assume that the vault context for Azure Site Recovery operations has been set.

## Validate that your Configuration Server and Scale out Process servers are registered to the vault

 Assume:
- A Configuration server named *ConfigurationServer* has been registered to this vault
- An additional Process Server named *ScaleOut-ProcessServer* has been registered to *ConfigurationServer*
- Accounts named *vCenter_account*, *WindowsAccount*, and *LinuxAccount* have been set up on the Configuration server. These accounts are used to add the vCenter server to discover virtual machines, and to push-install the mobility service software on Windows and Linux servers that are to be replicated.

Registered Configuration Servers are represented by a fabric object in Azure Site Recovery. In this step, get the list of fabric objects in the vault and identify the Configuration Server.

```azurepowershell
# Verify that the Configuration server is successfully registered to the vault
$ASRFabrics = Get-ASRFabric
$ASRFabrics.count
```
```
1
```
```azurepowershell
#Print details of the Configuration Server
$ASRFabrics[0]
```
```
Name                  : 2c33d710a5ee6af753413e97f01e314fc75938ea4e9ac7bafbf4a31f6804460d
FriendlyName          : ConfigurationServer
ID                    : /Subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/VMwareDRToAzurePs/providers/Microsoft.RecoveryServices/vaults/VMwareDRToAzurePs/replicationFabrics
                        /2c33d710a5ee6af753413e97f01e314fc75938ea4e9ac7bafbf4a31f6804460d
Type                  : Microsoft.RecoveryServices/vaults/replicationFabrics
FabricType            : VMware
SiteIdentifier        : ef7a1580-f356-4a00-aa30-7bf80f952510
FabricSpecificDetails : Microsoft.Azure.Commands.RecoveryServices.SiteRecovery.ASRVMWareSpecificDetails
```

* Identify the Process Servers that can be used to replicate machines.

```azurepowershell
$ProcessServers = $ASRFabrics[0].FabricSpecificDetails.ProcessServers
for($i=0; $i -lt $ProcessServers.count; $i++) {
 "{0,-5} {1}" -f $i, $ProcessServers[$i].FriendlyName
}
```
```
0     ScaleOut-ProcessServer
1     ConfigurationServer
```

From the output above ***$ProcessServers[0]*** corresponds to *ScaleOut-ProcessServer* and ***$ProcessServers[1]*** corresponds to the Process Server role on *ConfigurationServer*

* Identify accounts that have been set up on the Configuration Server.

```azurepowershell
$AccountHandles = $ASRFabrics[0].FabricSpecificDetails.RunAsAccounts
#Print the account details
$AccountHandles
```
```
AccountId AccountName
--------- -----------
1         vCenter_account
2         WindowsAccount
3         LinuxAccount
```

From the output above ***$AccountHandles[0]*** corresponds to the account *vCenter_account*, ***$AccountHandles[1]*** to account *WindowsAccount*, and ***$AccountHandles[2]*** to account *LinuxAccount*

## Create a replication policy and map it for use with the Configuration Server

In this step, two replication policies are created. One policy to replicate VMware virtual machines to Azure, and the other to replicate failed over virtual machines running in Azure back to the on-premises VMware site.

> [!NOTE]
> Most Azure Site Recovery operations are executed asynchronously. When you initiate an operation, an Azure Site Recovery job is submitted and a job tracking object is returned. This job tracking object can be used to monitor the status of the operation.

* Create a replication policy named *ReplicationPolicy* to replicate VMware virtual machines to Azure with the specified properties.

```azurepowershell
$Job_PolicyCreate = New-ASRPolicy -VMwareToAzure -Name "ReplicationPolicy" -RecoveryPointRetentionInHours 24 -ApplicationConsistentSnapshotFrequencyInHours 4 -RPOWarningThresholdInMinutes 60

# Track Job status to check for completion
while (($Job_PolicyCreate.State -eq "InProgress") -or ($Job_PolicyCreate.State -eq "NotStarted")){ 
        sleep 10; 
        $Job_PolicyCreate = Get-ASRJob -Job $Job_PolicyCreate
}

#Display job status
$Job_PolicyCreate
```
```
Name             : 8d18e2d9-479f-430d-b76b-6bc7eb2d0b3e
ID               : /Subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/VMwareDRToAzurePs/providers/Microsoft.RecoveryServices/vaults/VMwareDRToAzurePs/replicationJobs/8d18e2d
                   9-479f-430d-b76b-6bc7eb2d0b3e
Type             :
JobType          : AddProtectionProfile
DisplayName      : Create replication policy
ClientRequestId  : a162b233-55d7-4852-abac-3d595a1faac2 ActivityId: 9895234a-90ea-4c1a-83b5-1f2c6586252a
State            : Succeeded
StateDescription : Completed
StartTime        : 11/24/2017 2:49:24 AM
EndTime          : 11/24/2017 2:49:23 AM
TargetObjectId   : ab31026e-4866-5440-969a-8ebcb13a372f
TargetObjectType : ProtectionProfile
TargetObjectName : ReplicationPolicy
AllowedActions   :
Tasks            : {Prerequisites check for creating the replication policy, Creating the replication policy}
Errors           : {}
```

* Create a replication policy to use for failback from Azure to the on-premises VMware site.

```azurepowershell
$Job_FailbackPolicyCreate = New-ASRPolicy -AzureToVMware -Name "ReplicationPolicy-Failback" -RecoveryPointRetentionInHours 24 -ApplicationConsistentSnapshotFrequencyInHours 4 -RPOWarningThresholdInMinutes 60
```

Use the job details in *$Job_FailbackPolicyCreate* to track the operation to completion.

* Create a protection container mapping to map replication policies with the Configuration Server.

```azurepowershell
#Get the protection container corresponding to the Configuration Server
$ProtectionContainer = Get-ASRProtectionContainer -Fabric $ASRFabrics[0]

#Get the replication policies to map by name.
$ReplicationPolicy = Get-ASRPolicy -Name "ReplicationPolicy"
$FailbackReplicationPolicy = Get-ASRPolicy -Name "ReplicationPolicy-Failback"

# Associate the replication policies to the protection container corresponding to the Configuration Server. 

$Job_AssociatePolicy = New-ASRProtectionContainerMapping -Name "PolicyAssociation1" -PrimaryProtectionContainer $ProtectionContainer -Policy $Policy[0]

# Check the job status
while (($Job_AssociatePolicy.State -eq "InProgress") -or ($Job_AssociatePolicy.State -eq "NotStarted")){ 
        sleep 10; 
        $Job_AssociatePolicy = Get-ASRJob -Job $Job_AssociatePolicy
}
$Job_AssociatePolicy.State

$Job_AssociateFailbackPolicy = New-ASRProtectionContainerMapping -Name "FailbackPolicyAssociation" -PrimaryProtectionContainer $ProtectionContainer -Policy $Policy[0]

# Check the job status
while (($Job_AssociateFailbackPolicy.State -eq "InProgress") -or ($Job_AssociateFailbackPolicy.State -eq "NotStarted")){ 
        sleep 10; 
        $Job_AssociateFailbackPolicy = Get-ASRJob -Job $Job_AssociateFailbackPolicy
}
$Job_AssociateFailbackPolicy.State

```

## Add a vCenter server and discover VMs

Add a vCenter Server by IP address or hostname. The **-port** parameter specifies the port on the vCenter server to connect to, **-Name** parameter specifies a friendly name to use for the vCenter server, and the  **-Account** parameter specifies the account handle on the Configuration server to use to discover virtual machines managed by the vCenter server.

```azurepowershell
# The $AccountHandles[0] variable holds details of vCenter_account

$Job_AddvCenterServer = New-ASRvCenter -Fabric $ASRFabrics[0] -Name "MyvCenterServer" -IpOrHostName "10.150.24.63" -Account $AccountHandles[0] -Port 443

#Wait for the job to complete and ensure it completed successfully

while (($Job_AddvCenterServer.State -eq "InProgress") -or ($Job_AddvCenterServer.State -eq "NotStarted")) { 
        sleep 30; 
        $Job_AddvCenterServer = Get-ASRJob -Job $Job_AddvCenterServer
}
$Job_AddvCenterServer
```
```
Name             : 0f76f937-f9cf-4e0e-bf27-10c9d1c252a4
ID               : /Subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/VMwareDRToAzurePs/providers/Microsoft.RecoveryServices/vaults/VMwareDRToAzurePs/replicationJobs/0f76f93
                   7-f9cf-4e0e-bf27-10c9d1c252a4
Type             :
JobType          : DiscoverVCenter
DisplayName      : Add vCenter server
ClientRequestId  : a2af8892-5686-4d64-a528-10445bc2f698 ActivityId: 7ec05aad-002e-4da0-991f-95d0de7a9f3a
State            : Succeeded
StateDescription : Completed
StartTime        : 11/24/2017 2:41:47 AM
EndTime          : 11/24/2017 2:44:37 AM
TargetObjectId   : 10.150.24.63
TargetObjectType : VCenter
TargetObjectName : MyvCenterServer
AllowedActions   :
Tasks            : {Adding vCenter server}
Errors           : {}
```

## Create storage accounts

In this step, the storage accounts to be used for replication are created. These storage accounts are used later to replicate virtual machines. Ensure that the storage accounts are created in the same Azure region as the vault. You can skip this step if you plan to use an existing storage account for replication.

> [!NOTE]
> While replicating on-premises virtual machines to a premium storage account, you need to specify an additional standard storage account (log storage account). The log storage account  holds replication logs as an intermediate store until the logs can be applied on the premium storage target.
>

```azurepowershell

$PremiumStorageAccount = New-AzureRmStorageAccount -ResourceGroupName "VMwareDRToAzurePs" -Name "premiumstorageaccount1" -Location "East Asia" -SkuName Premium_LRS

$LogStorageAccount = New-AzureRmStorageAccount -ResourceGroupName "VMwareDRToAzurePs" -Name "logstorageaccount1" -Location "East Asia" -SkuName Standard_LRS

$ReplicationStdStorageAccount= New-AzureRmStorageAccount -ResourceGroupName "VMwareDRToAzurePs" -Name "replicationstdstorageaccount1" -Location "East Asia" -SkuName Standard_LRS
```

## Replicate VMware virtual machines

It takes about 15-20 minutes for virtual machines to be discovered from the vCenter server. Once discovered, a protectable item object is created in Azure Site Recovery for each discovered virtual machine. In this step, three of the discovered virtual machines are replicated to the Azure Storage accounts created in the previous step.

You will need the following details to protect a discovered virtual machine:
* The protectable item to be replicated.
* The storage account to replicate the virtual machine to. Additionally, a log storage is needed to protect virtual machines to a premium storage account.
* The Process Server to be used for replication. The list of available process servers has been retrieved and saved in the ***$ProcessServers[0]***  *(ScaleOut-ProcessServer)* and ***$ProcessServers[1]*** *(ConfigurationServer)* variables.
* The account to use to push-install the Mobility service software onto the machines. The list of available accounts has been retrieved and stored in the ***$AccountHandles*** variable.
* The protection container mapping for the replication policy to be used for replication.
* The resource group in which virtual machines must be created on failover.
* Optionally, the Azure virtual network and subnet to which the failed over virtual machine should be connected.

Now replicate the following virtual machines using the settings specified in this table


|Virtual machine  |Process Server        |Storage Account              |Log Storage account  |Policy           |Account for Mobility service installation|Target resource group  | Target virtual network  |Target subnet  |
|-----------------|----------------------|-----------------------------|---------------------|-----------------|-----------------------------------------|-----------------------|-------------------------|---------------|
|Win2K12VM1       |ScaleOut-ProcessServer|premiumstorageaccount1       |logstorageaccount1   |ReplicationPolicy|WindowsAccount                           |VMwareDRToAzurePs      |ASR-vnet                 |Subnet-1       |
|CentOSVM1       |ConfigurationServer   |replicationstdstorageaccount1| N/A                 |ReplicationPolicy|LinuxAccount                             |VMwareDRToAzurePs      |ASR-vnet                 |Subnet-1       |   
|CentOSVM2       |ConfigurationServer   |replicationstdstorageaccount1| N/A                 |ReplicationPolicy|LinuxAccount                             |VMwareDRToAzurePs      |ASR-vnet                 |Subnet-1       |   

 
```azurepowershell

#Get the target resource group to be used
$ResourceGroup = Get-AzureRmResourceGroup -Name "VMwareToAzureDrPs"

#Get the target virtual network to be used
$RecoveryVnet = Get-AzureRmVirtualNetwork -Name "ASR-vnet" -ResourceGroupName "asrrg" 

#Get the protection container mapping for replication policy named ReplicationPolicy
$PolicyMap  = Get-ASRProtectionContainerMapping -ProtectionContainer $ProtectionContainer | where PolicyFriendlyName -eq "ReplicationPolicy"

#Get the protectable item corresponding to the virtual machine Win2K12VM1
$VM1 = Get-ASRProtectableItem -ProtectionContainer $ProtectionContainer -FriendlyName "Win2K12VM1"

# Enable replication for virtual machine Win2K12VM1
# The name specified for the replicated item needs to be unique within the protection container. Using a random GUID to ensure uniqueness
$Job_EnableRepication1 = New-ASRReplicationProtectedItem -VMwareToAzure -ProtectableItem $VM1 -Name (New-Guid).Guid -ProtectionContainerMapping $PolicyMap -RecoveryAzureStorageAccountId $PremiumStorageAccount.Id -LogStorageAccountId $LogStorageAccount.Id -ProcessServer $ProcessServers[0] -Account $AccountHandles[1] -RecoveryResourceGroupId $ResourceGroup.ResourceId -RecoveryAzureNetworkId $RecoveryVnet.Id -RecoveryAzureSubnetName "Subnet-1" 

#Get the protectable item corresponding to the virtual machine CentOSVM1
$VM2 = Get-ASRProtectableItem -ProtectionContainer $ProtectionContainer -FriendlyName "CentOSVM1"

# Enable replication for virtual machine CentOSVM1
$Job_EnableRepication2 = New-ASRReplicationProtectedItem -VMwareToAzure -ProtectableItem $VM2 -Name (New-Guid).Guid -ProtectionContainerMapping $PolicyMap -RecoveryAzureStorageAccountId $ReplicationStdStorageAccount.Id  -ProcessServer $ProcessServers[1] -Account $AccountHandles[2] -RecoveryResourceGroupId $ResourceGroup.ResourceId -RecoveryAzureNetworkId $RecoveryVnet.Id -RecoveryAzureSubnetName "Subnet-1"

#Get the protectable item corresponding to the virtual machine CentOSVM2
$VM3 = Get-ASRProtectableItem -ProtectionContainer $ProtectionContainer -FriendlyName "CentOSVM2"

# Enable replication for virtual machine CentOSVM2
$Job_EnableRepication3 = New-ASRReplicationProtectedItem -VMwareToAzure -ProtectableItem $VM3 -Name (New-Guid).Guid -ProtectionContainerMapping $PolicyMap -RecoveryAzureStorageAccountId $ReplicationStdStorageAccount.Id  -ProcessServer $ProcessServers[1] -Account $AccountHandles[2] -RecoveryResourceGroupId $ResourceGroup.ResourceId -RecoveryAzureNetworkId $RecoveryVnet.Id -RecoveryAzureSubnetName "Subnet-1" 

```

Once the enable replication job completes successfully, initial replication is started for the virtual machines. Initial replication may take a while depending on the amount of data to be replicated and the bandwidth available for replication. After initial replication completes, the virtual machine moves to a protected state. Once the virtual machine reaches a protected state you can perform a test failover for the virtual machine, add it to recovery plans etc.

You can check the replication state and replication health of the virtual machine with the Get-ASRReplicationProtectedItem cmdlet.

```azurepowershell
Get-ASRReplicationProtectedItem -ProtectionContainer $ProtectionContainer | Select FriendlyName, ProtectionState, ReplicationHealth
```
```
FriendlyName ProtectionState                 ReplicationHealth
------------ ---------------                 -----------------
CentOSVM1    Protected                       Normal
CentOSVM2    InitialReplicationInProgress    Normal
Win2K12VM1   Protected                       Normal
```

## Configure failover settings for replicating virtual machines

Failover settings for protected machines can be updated using the Set-ASRReplicationProtectedItem cmdlet. Some of the settings that can be updated through this cmdlet are:
* Name of the virtual machine to be created on failover
* VM size of the virtual machine to be created on failover
* Azure virtual network and subnet that the NICs of the virtual machine should be connected to on failover
* Failover to managed disks
* Apply Azure Hybrid Use Benefit
* Assign a static IP address from the target virtual network to be assigned to the virtual machine on failover.

In this example, we update the VM size of the virtual machine to be created on failover for the virtual machine *Win2K12VM1* and specify that the virtual machine use managed disks on failover.

```azurepowershell
$ReplicatedVM1 = Get-ASRReplicationProtectedItem -FriendlyName "Win2K12VM1" -ProtectionContainer $ProtectionContainer

Set-ASRReplicationProtectedItem -InputObject $ReplicatedVM1 -Size "Standard_DS11" -UseManagedDisk True
```
```
Name             : cafa459c-44a7-45b0-9de9-3d925b0e7db9
ID               : /Subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/VMwareDRToAzurePs/providers/Microsoft.RecoveryServices/vaults/VMwareDRToAzurePs/replicationJobs/cafa459
                   c-44a7-45b0-9de9-3d925b0e7db9
Type             :
JobType          : UpdateVmProperties
DisplayName      : Update the virtual machine
ClientRequestId  : b0b51b2a-f151-4e9a-a98e-064a5b5131f3 ActivityId: ac2ba316-be7b-4c94-a053-5363f683d38f
State            : InProgress
StateDescription : InProgress
StartTime        : 11/24/2017 2:04:26 PM
EndTime          :
TargetObjectId   : 88bc391e-d091-11e7-9484-000c2955bb50
TargetObjectType : ProtectionEntity
TargetObjectName : Win2K12VM1
AllowedActions   :
Tasks            : {Update the virtual machine properties}
Errors           : {}
```

## Perform a test failover, validate, and cleanup test failover

```azurepowershell
#Test failover of Win2K12VM1 to the test virtual network "V2TestNetwork"

#Get details of the test failover virtual network to be used
TestFailovervnet = Get-AzureRmVirtualNetwork -Name "V2TestNetwork" -ResourceGroupName "asrrg" 

#Start the test failover operation
$TFOJob = Start-ASRTestFailoverJob -ReplicationProtectedItem $ReplicatedVM1 -AzureVMNetworkId $TestFailovervnet.Id -Direction PrimaryToRecovery
```
Once the test failover job completes successfully, you will notice that a virtual machine suffixed with *"-Test"* (Win2K12VM1-Test in this case) to its name is created in Azure. 

You can now connect to the test failed over virtual machine, and validate the test failover.

Cleanup test failover using the Start-ASRTestFailoverCleanupJob cmdlet. This operation deletes the virtual machine created as part of the test failover operation.

```azurepowershell
$Job_TFOCleanup = Start-ASRTestFailoverCleanupJob -ReplicationProtectedItem $ReplicatedVM1
```

## Failover to Azure

In this step, we failover the virtual machine Win2K12VM1 to a specific recovery point.

```azurepowershell
# Get the list of available recovery points for Win2K12VM1
$RecoveryPoints = Get-ASRRecoveryPoint -ReplicationProtectedItem $ReplicatedVM1
"{0} {1}" -f $RecoveryPoints[0].RecoveryPointType, $RecoveryPoints[0].RecoveryPointTime
```
```
CrashConsistent 11/24/2017 5:28:25 PM
```
```azurepowershell

#Start the failover job
$Job_Failover = Start-ASRUnplannedFailoverJob -ReplicationProtectedItem $ReplicatedVM1 -Direction PrimaryToRecovery -RecoveryPoint $RecoveryPoints[0]
do {
        $Job_Failover = Get-ASRJob -Job $Job_Failover;
        sleep 60;
} while (($Job_Failover.State -eq "InProgress") -or ($JobFailover.State -eq "NotStarted"))
$Job_Failover.State
```
```
Succeeded
```

Once failed over successfully, you can commit the failover operation and setup reverse replication from Azure back to the on-premises VMware site.

## Next steps
View the [Azure Site Recovery PowerShell reference ](https://docs.microsoft.com/powershell/module/AzureRM.RecoveryServices.SiteRecovery) to learn how you can perform other tasks such as creating Recovery Plans and testing failover of Recovery plans through PowerShell.
