# Azure Database Migration Project


## Process

### Setup the Production Environment
 - Provisioned a virtual machine `DM-Project-VM` in the resource group `ai_core_vm_rg` [originally this was chosen as 'Standard_B1s' with a single vCPU as it's free tier eligible, however this was too slow to use for installing SMSS, so VM was resized to 'Standard_B2s']
 - Installed SQL Server on `DM-Project-VM`
 - Installed SQL Server Management Studio (SMSS) on `DM-Project-VM`
 - Downloaded the `AdventureWorks.bak` database backup file
 - Restored the Adventure Works database from backup
