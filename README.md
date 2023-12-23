# Azure Database Migration Project


## Process

### Setup the Production Environment
 - Provisioned a virtual machine `DM-Project-VM` in the resource group `ai_core_vm_rg` [originally this was chosen as 'Standard_B1s' with a single vCPU as it's free tier eligible, however this was too slow to use for installing SMSS, so VM was resized to 'Standard_B2s']
 - Installed SQL Server on `DM-Project-VM`
 - Installed SQL Server Management Studio (SMSS) on `DM-Project-VM`
 - Downloaded the `AdventureWorks.bak` database backup file
 - Restored the Adventure Works database from backup

### Migrate to Azure SQL Database
 - Create a SQL database on azure: `DM-Project-Database` located on the server `my-sql-server-6432`

### Data Backup and Restore
 - Made a backup of the AdventureWorks database on the production virtual machine `DM-Project-VM` (saved to the default location in Program Files )
 - Created a new storage account in Azure dashboard `dmprojectstore`
 - Created a new container within `dmprojectstore` called `dm-project-backup-container` to house backups for the production database
 - Uploaded the database backup to the storage container using the Azure dashboard
 - Created a new deveploment environment virtual machine `DM-Project-DevEnvVM` [note: the free trial of azure only extends to 4 vCPUs in total so had to downsize the original production VM to 'Standard D2ds v5' and used the same for this new dev environemnt VM]
 - Installed SQL Server and SMSS on this dev environment VM
 - Restored the AdventureWorks database from the backup stored in `dm-project-backup-container`
 - Created a new container `dm-project-dev-env-backup` in `dmprojectstore` for storing backups of this dev environment database
 - Created credentials in SMSS on `DM-Project-DevEnvVM` for access to the newly created storage container [labelled as `dev-env-vm-cred`]
 - Created a maintenance plan to backup this database to `dm-project-dev-env-backup`
 - Executed maintenance plan and checked it stored the appropriate .bak files in the container
 - Setup a weekly plan for backup of the database

### Disaster Recovery Simulation
 - Query the database:
```
SELECT * FROM Person.Address
WHERE City = 'Seattle';
```
- Delete these records to simluate data loss:
```
DELETE FROM Person.Address
WHERE City = 'Seattle';
```
- Confirm these records have been deleted by running the initial query again
- Restored the Azure SQL Database from a previous point in time (one day ago selected)
- Connect to the new restored database
- Run the intial query to confirm the deleted data has been restored
- Use this restored database as the new production database and delete the previous one via the Azure dashboard
