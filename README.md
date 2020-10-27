Create Azure Windows 10 Creator VM using PowerShell
===================================================

            
This Script was created to install Azure Win. 10 VM with Resource group including storage account, virtual network & storage content. The below script is for installing Azure VM for Windows 10 pro
 last version.

 



PowerShell
Edit|Remove
powershell
##########################################################################################################
<#
.SYNOPSIS
    Create Lab resource group, storage account, virtual network and VMs
    
.DESCRIPTION
    This script will create the following components
    -	Resource group: it will contain all VMs, storage account, virtual network and other resources required for the lab.
    -   You can edit on labPrefix, labnumber, labsubnet & Ip address with name as you want
    -   You can edit on Virtual machine size, Virtual machine name, NicName with name as you want
    -	Windows 10 Client VM.
    -	Username: .\labadmin
    -	Password: Passw0rd


.NOTES
    THIS CODE-SAMPLE IS PROVIDED 'AS IS' WITHOUT WARRANTY OF ANY KIND, EITHER EXPRESSED 
    OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE IMPLIED WARRANTIES OF MERCHANTABILITY AND/OR 
    FITNESS FOR A PARTICULAR PURPOSE.

    

#>
##########################################################################################################
###############Install Azure Module#########################
Install-Module -Name AzureRM -force
Import-Module -Name AzureRM
Install-Module -Name AzureRM.compute
Import-Module -Name AzureRM.compute
# Connect to Azure

Login-AzurermAccount

# Select Azure Subscription
$sub = Get-AzureRMsubscription
Select-AzureRmSubscription -SubscriptionId $sub[0].SubscriptionId

# Set values for new resource group, storage account and vnet
$labPrefix = 'Mlab'
$labnumber = '2017'
$labsubnet = '55'
$rgName = $labPrefix + $labnumber #New resource group name
$locName = 'West Europe' # Loation of new resource group
$saName = $rgName.Replace('-','').tolower() # Storage account name (for new resource group)
$saType='Standard_LRS' # Storage account type
$vnetname = $rgName # New virtual network name
$subnetIndex=0 # Frontend subnet ID
$frontendsubnetname = $rgName + '-FE' # Frontend subnet name
$backendsubnetname = $rgName + '-BE' # Backtend subnet name
$vnetsubnet = '10.$labsubnet.0.0/16' # Virtual network suffix
$frontendsubnetrange = '10.$labsubnet.1.0/24' # Frontend subnet range
$backendsubnetrange = '10.$labsubnet.2.0/24' # Backtend subnet name
$NSGName = $rgName + '-NSG'


#VM2 Config
$domName2= ($rgName + '-cl').tolower()
$nicName2=($rgName + '-cl').tolower()
$staticIP2 = '10.$labsubnet.1.101'
$vmName2= ($rgName + '-cl').tolower()
$vmSize2='Standard_A2'
$vm2 = New-AzureRmVMConfig -VMName $vmName2 -VMSize $vmSize2

## Create initial resources
# create resource group
New-AzureRmResourceGroup -Name $rgName -Location $locName | Out-Null
write-output '1/14 - the resource group has been created successfully'
# create storage account
New-AzureRmStorageAccount -Name $saName -ResourceGroupName $rgName –Type $saType -Location $locName | Out-Null
write-output '2/14 - the storage account has been created successfully'
# get storage account key
$key1 = (Get-AzureRmStorageAccountKey -Name $saName -ResourceGroupName $rgName).Value[0]
# create storage context
$storagecontext = New-AzureStorageContext -StorageAccountName $saname -StorageAccountKey $key1
# create a container called scripts
New-AzureStorageContainer -Name 'scripts' -Context $storagecontext -Permission BLOB  | Out-Null
write-output '3/14 - Scripts Container has been created successfully'
New-AzureStorageContainer -Name 'baseimage' -Context $storagecontext -Permission BLOB | Out-Null
write-output '4/14 - Base Image Container has been created successfully'


########################################################## Virtual network config ##############################################################

$frontendSubnet=New-AzureRmVirtualNetworkSubnetConfig -Name $frontendsubnetname -AddressPrefix $frontendsubnetrange
$backendSubnet=New-AzureRmVirtualNetworkSubnetConfig -Name $backendsubnetname -AddressPrefix $backendsubnetrange
$vnet = New-AzureRmVirtualNetwork -Name $vnetname -ResourceGroupName $rgName -Location $locName -AddressPrefix $vnetsubnet -Subnet $frontendSubnet,$backendsubnet -DnsServer $staticIP1,8.8.8.8
$rule1 = New-AzureRmNetworkSecurityRuleConfig -Name rdp-rule -Description 'Allow RDP' -Access Allow -Protocol Tcp -Direction Inbound -Priority 100 -SourceAddressPrefix Internet -SourcePortRange * -DestinationAddressPrefix * -DestinationPortRange 3389
$rule2 = New-AzureRmNetworkSecurityRuleConfig -Name remoteps-rule -Description 'Allow remote powershell' -Access Allow -Protocol Tcp -Direction Inbound -Priority 101 -SourceAddressPrefix Internet -SourcePortRange * -DestinationAddressPrefix * -DestinationPortRange 5986
$rule3 = New-AzureRmNetworkSecurityRuleConfig -Name winrm-rule -Description 'Allow winrm' -Access Allow -Protocol Tcp -Direction Inbound -Priority 102 -SourceAddressPrefix Internet -SourcePortRange * -DestinationAddressPrefix * -DestinationPortRange 5985
$rule4 = New-AzureRmNetworkSecurityRuleConfig -Name SMTP-rule -Description 'SMTP' -Access Allow -Protocol Tcp -Direction Inbound -Priority 103 -SourceAddressPrefix Internet -SourcePortRange * -DestinationAddressPrefix * -DestinationPortRange 25
$rule5 = New-AzureRmNetworkSecurityRuleConfig -Name HTTP-rule -Description 'HTTP' -Access Allow -Protocol Tcp -Direction Inbound -Priority 104 -SourceAddressPrefix Internet -SourcePortRange * -DestinationAddressPrefix * -DestinationPortRange 80
$rule6 = New-AzureRmNetworkSecurityRuleConfig -Name HTTPS-rule -Description 'HTTPS' -Access Allow -Protocol Tcp -Direction Inbound -Priority 105 -SourceAddressPrefix Internet -SourcePortRange * -DestinationAddressPrefix * -DestinationPortRange 443
$nsg = New-AzureRmNetworkSecurityGroup -ResourceGroupName $rgName -Location $locName -Name $NSGName -SecurityRules $rule1,$rule2,$rule3,$rule4,$rule5,$rule6
Set-AzureRmVirtualNetworkSubnetConfig -VirtualNetwork $vnet -Name $frontendsubnetname -AddressPrefix $frontendsubnetrange -NetworkSecurityGroup $nsg | Out-Null
Set-AzureRmVirtualNetwork -VirtualNetwork $vnet | Out-Null
$vnet = Get-AzureRMVirtualNetwork -Name $vnetName -ResourceGroupName $rgName
write-output '8/14 - Virtual netwrok has been created successfully with two subnets frontend & backend. Also NSG has been created successfully'


########################################################## Windows 10 client ###################################################################
write-output 'Creating Client VM, please wait'
$pip2=New-AzureRmPublicIpAddress -Name $nicName2 -ResourceGroupName $rgName -DomainNameLabel $domName2 -Location $locName -AllocationMethod Dynamic
$nic2=New-AzureRmNetworkInterface -Name $nicName2 -ResourceGroupName $rgName -Location $locName -SubnetId $vnet.Subnets[$subnetIndex].Id -PublicIpAddressId $pip2.Id -PrivateIpAddress $staticIP2
$pubName2='MicrosoftWindowsDesktop'
$offerName2='Windows-10'
$skuName2='RS3-Pro'
$username = 'labadmin'
$password = 'Passw0rd' | ConvertTo-SecureString -AsPlainText -Force
$cred = New-Object -typename System.Management.Automation.PSCredential -argumentlist $username, $password
$vm2 =Set-AzurermVMOperatingSystem -VM $vm2 -Windows -ComputerName $vmName2 -Credential $cred -ProvisionVMAgent -EnableAutoUpdate
$vm2 =Set-AzurermVMSourceImage -VM $vm2 -PublisherName $pubName2 -Offer $offerName2 -Skus $skuName2 -Version 'latest'
$vm2 = Add-AzurermVMNetworkInterface -VM $vm2 -Id $nic2.Id
$diskName='OSDisk'
$storageAcc=Get-AzureRmStorageAccount -ResourceGroupName $rgName -Name $saName
$osDiskUri=$storageAcc.PrimaryEndpoints.Blob.ToString() + 'vhds/' + $vmName2 + $diskName  + '.vhd'
$vm2=Set-AzureRmVMOSDisk -VM $vm2 -Name $diskName -VhdUri $osDiskUri -CreateOption fromImage -Windows
New-AzureRmVM -ResourceGroupName $rgName -Location $locName -VM $vm2 | Out-Null
write-output 'Client VM has been created successfully, applying VM customization'



##########################################################################################################<# 
.SYNOPSIS 
    Create Lab resource group, storage account, virtual network and VMs 
     
.DESCRIPTION 
    This script will create the following components 
    -    Resource group: it will contain all VMs, storage account, virtual network and other resources required for the lab. 
    -   You can edit on labPrefix, labnumber, labsubnet & Ip address with name as you want 
    -   You can edit on Virtual machine size, Virtual machine name, NicName with name as you want 
    -    Windows 10 Client VM. 
    -    Username: .\labadmin 
    -    Password: Passw0rd 
 
 
.NOTES 
    THIS CODE-SAMPLE IS PROVIDED 'AS IS' WITHOUT WARRANTY OF ANY KIND, EITHER EXPRESSED  
    OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE IMPLIED WARRANTIES OF MERCHANTABILITY AND/OR  
    FITNESS FOR A PARTICULAR PURPOSE. 
 
     
 
#>#########################################################################################################################Install Azure Module######################### 
Install-Module -Name AzureRM -force 
Import-Module -Name AzureRM 
Install-Module -Name AzureRM.compute 
Import-Module -Name AzureRM.compute 
# Connect to Azure 
 
Login-AzurermAccount 
 
# Select Azure Subscription$sub = Get-AzureRMsubscription 
Select-AzureRmSubscription -SubscriptionId $sub[0].SubscriptionId 
 
# Set values for new resource group, storage account and vnet$labPrefix = 'Mlab'$labnumber = '2017'$labsubnet = '55'$rgName = $labPrefix+$labnumber#New resource group name$locName = 'West Europe'# Loation of new resource group$saName = $rgName.Replace('-','').tolower() # Storage account name (for new resource group)$saType='Standard_LRS'# Storage account type$vnetname = $rgName# New virtual network name$subnetIndex=0 # Frontend subnet ID$frontendsubnetname = $rgName+'-FE'# Frontend subnet name$backendsubnetname = $rgName+'-BE'# Backtend subnet name$vnetsubnet = '10.$labsubnet.0.0/16'# Virtual network suffix$frontendsubnetrange = '10.$labsubnet.1.0/24'# Frontend subnet range$backendsubnetrange = '10.$labsubnet.2.0/24'# Backtend subnet name$NSGName = $rgName+'-NSG'#VM2 Config$domName2= ($rgName+'-cl').tolower() 
$nicName2=($rgName+'-cl').tolower() 
$staticIP2 = '10.$labsubnet.1.101'$vmName2= ($rgName+'-cl').tolower() 
$vmSize2='Standard_A2'$vm2 = New-AzureRmVMConfig -VMName $vmName2-VMSize $vmSize2## Create initial resources# create resource group 
New-AzureRmResourceGroup -Name $rgName-Location $locName|Out-Null 
write-output '1/14 - the resource group has been created successfully'# create storage account 
New-AzureRmStorageAccount -Name $saName-ResourceGroupName $rgName –Type$saType-Location $locName|Out-Null 
write-output '2/14 - the storage account has been created successfully'# get storage account key$key1 = (Get-AzureRmStorageAccountKey -Name $saName-ResourceGroupName $rgName).Value[0] 
# create storage context$storagecontext = New-AzureStorageContext -StorageAccountName $saname-StorageAccountKey $key1# create a container called scripts 
New-AzureStorageContainer -Name 'scripts'-Context $storagecontext-Permission BLOB  |Out-Null 
write-output '3/14 - Scripts Container has been created successfully' 
New-AzureStorageContainer -Name 'baseimage'-Context $storagecontext-Permission BLOB |Out-Null 
write-output '4/14 - Base Image Container has been created successfully'########################################################## Virtual network config ##############################################################$frontendSubnet=New-AzureRmVirtualNetworkSubnetConfig -Name $frontendsubnetname-AddressPrefix $frontendsubnetrange$backendSubnet=New-AzureRmVirtualNetworkSubnetConfig -Name $backendsubnetname-AddressPrefix $backendsubnetrange$vnet = New-AzureRmVirtualNetwork -Name $vnetname-ResourceGroupName $rgName-Location $locName-AddressPrefix $vnetsubnet-Subnet $frontendSubnet,$backendsubnet-DnsServer $staticIP1,8.8.8.8 
$rule1 = New-AzureRmNetworkSecurityRuleConfig -Name rdp-rule -Description 'Allow RDP'-Access Allow -Protocol Tcp -Direction Inbound -Priority 100 -SourceAddressPrefix Internet -SourcePortRange *-DestinationAddressPrefix *-DestinationPortRange 3389 
$rule2 = New-AzureRmNetworkSecurityRuleConfig -Name remoteps-rule -Description 'Allow remote powershell'-Access Allow -Protocol Tcp -Direction Inbound -Priority 101 -SourceAddressPrefix Internet -SourcePortRange *-DestinationAddressPrefix *-DestinationPortRange 5986 
$rule3 = New-AzureRmNetworkSecurityRuleConfig -Name winrm-rule -Description 'Allow winrm'-Access Allow -Protocol Tcp -Direction Inbound -Priority 102 -SourceAddressPrefix Internet -SourcePortRange *-DestinationAddressPrefix *-DestinationPortRange 5985 
$rule4 = New-AzureRmNetworkSecurityRuleConfig -Name SMTP-rule -Description 'SMTP'-Access Allow -Protocol Tcp -Direction Inbound -Priority 103 -SourceAddressPrefix Internet -SourcePortRange *-DestinationAddressPrefix *-DestinationPortRange 25 
$rule5 = New-AzureRmNetworkSecurityRuleConfig -Name HTTP-rule -Description 'HTTP'-Access Allow -Protocol Tcp -Direction Inbound -Priority 104 -SourceAddressPrefix Internet -SourcePortRange *-DestinationAddressPrefix *-DestinationPortRange 80 
$rule6 = New-AzureRmNetworkSecurityRuleConfig -Name HTTPS-rule -Description 'HTTPS'-Access Allow -Protocol Tcp -Direction Inbound -Priority 105 -SourceAddressPrefix Internet -SourcePortRange *-DestinationAddressPrefix *-DestinationPortRange 443 
$nsg = New-AzureRmNetworkSecurityGroup -ResourceGroupName $rgName-Location $locName-Name $NSGName-SecurityRules $rule1,$rule2,$rule3,$rule4,$rule5,$rule6 
Set-AzureRmVirtualNetworkSubnetConfig -VirtualNetwork $vnet-Name $frontendsubnetname-AddressPrefix $frontendsubnetrange-NetworkSecurityGroup $nsg|Out-Null 
Set-AzureRmVirtualNetwork -VirtualNetwork $vnet|Out-Null$vnet = Get-AzureRMVirtualNetwork -Name $vnetName-ResourceGroupName $rgName 
write-output '8/14 - Virtual netwrok has been created successfully with two subnets frontend & backend. Also NSG has been created successfully'########################################################## Windows 10 client ################################################################### 
write-output 'Creating Client VM, please wait'$pip2=New-AzureRmPublicIpAddress -Name $nicName2-ResourceGroupName $rgName-DomainNameLabel $domName2-Location $locName-AllocationMethod Dynamic 
$nic2=New-AzureRmNetworkInterface -Name $nicName2-ResourceGroupName $rgName-Location $locName-SubnetId $vnet.Subnets[$subnetIndex].Id -PublicIpAddressId $pip2.Id -PrivateIpAddress $staticIP2$pubName2='MicrosoftWindowsDesktop'$offerName2='Windows-10'$skuName2='RS3-Pro'$username = 'labadmin'$password = 'Passw0rd'|ConvertTo-SecureString-AsPlainText -Force 
$cred = New-Object-typename System.Management.Automation.PSCredential -argumentlist $username, $password$vm2 =Set-AzurermVMOperatingSystem -VM $vm2-Windows -ComputerName $vmName2-Credential $cred-ProvisionVMAgent -EnableAutoUpdate 
$vm2 =Set-AzurermVMSourceImage -VM $vm2-PublisherName $pubName2-Offer $offerName2-Skus $skuName2-Version 'latest'$vm2 = Add-AzurermVMNetworkInterface -VM $vm2-Id $nic2.Id 
$diskName='OSDisk'$storageAcc=Get-AzureRmStorageAccount -ResourceGroupName $rgName-Name $saName$osDiskUri=$storageAcc.PrimaryEndpoints.Blob.ToString() +'vhds/'+$vmName2+$diskName+'.vhd'$vm2=Set-AzureRmVMOSDisk -VM $vm2-Name $diskName-VhdUri $osDiskUri-CreateOption fromImage -Windows 
New-AzureRmVM -ResourceGroupName $rgName-Location $locName-VM $vm2|Out-Null 
write-output 'Client VM has been created successfully, applying VM customization'




 


You can use this script in following steps:    



  *  Download the script and copy it to your Server or Client machine.

  *  Run Windows PowerShell as Administrator 
  *  .\CreateAzureWin10.ps1. 
**Note:**


  *  You will need to install Azure PowerShell before running this script.

  *  You can edit on labPrefix, labnumber, labsubnet & Ip address with name as you want

  *  You can edit on Virtual machine size, Virtual machine name, NicName with name as you want


    
TechNet gallery is retiring! This script was migrated from TechNet script center to GitHub by Microsoft Azure Automation product group. All the Script Center fields like Rating, RatingCount and DownloadCount have been carried over to Github as-is for the migrated scripts only. Note : The Script Center fields will not be applicable for the new repositories created in Github & hence those fields will not show up for new Github repositories.
