# Azure Database Migration Project


## Process

### Setup the Production Environment
 - Provisioned a virtual machine `DM-Project-VM` in the resource group `ai_core_vm_rg` [originally this was chosen as 'Standard_B1s' with a single vCPU as it's free tier eligible, however this was too slow to use for installing SMSS, so VM was resized to 'Standard_B2s']
 - Installed SQL Server on `DM-Project-VM`
 - Installed SQL Server Management Studio (SMSS) on `DM-Project-VM`
 - Downloaded the `AdventureWorks.bak` database backup file
 - Restored the Adventure Works database from backup

### Migrate to Azure SQL Database
 - Created a SQL database on azure: `DM-Project-Database` located on the server `my-sql-server-6432` which uses SQL authentication
 - Opened Azure Data Studio on `DM-Project-VM` to create a connection to the SQL database (using the 'connection string' from the Azure SQL database dashboard to quickly input most of the details)
 -  Installed SQL Server Schema Compare extension
 -  Right click on the connected database to get the 'Schema Compare' pane open and select the localhost as source and remote server on Azure as target
 -  Applied the schema compare to update changes to the target and confirmed the table names appeared under the connected database
 -  Installed Azure SQL Migration extension
 -  Followed the Migrate to Azure SQL wizard to migrate the localhost database to the Azure SQL server (which requires installation of Integration Runtime) [note: this was attempted 3 times on the current VM configuration and would result in the VM hanging ((azure studio would crash, then integration runtime would crash, and upon reloading everything the connection to azure account would need to be reconnected), after upgrading the VM once again to a higher tier ('Standard D4ds v5') it worked first time]
 - Confirmed the data had transferred by runnning queries on localhost and Azure SQL server which returned the same results

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

### Geo Replication and Failover
- Set up a new SQL server on Azure based in a different geographical region (US East) called `my-replication-server-6432`
- In the production SQL database set up a replica to this newly created geo replication server
- On production SQL server `my-sql-server-6432` set up a failover group with the geo replication server as secondary
- On the secondary server check that it is in the failover group, and select that group
- Initiate a failover using the 'Failover' button in Azure dashboard
- Check that failover has completed successfully by connecting to the database in the geo replication server using Azure Data Studio and then running a few queries to confirm database contents and functionality, e.g. run this query again to check for the same 141 records:
```
SELECT * FROM Person.Address
WHERE City = 'Seattle';
```
- Initiate failover again to failback to the original configuration of primary and secondary servers

### Microsoft Entra Directory Integration
- On production SQL server `my-sql-server-6432` under the Microsoft Entra ID tab select set admin to assign an admin
- From Azure Data Studio on the production VM disconnect from the SQL server and edit the connection settings to the new Entra ID login rather than SQL authentication
- Test the admin login works with an SQL query
- Under the Microsoft Entra ID section accessed from Azure dashboard create a new user `DB_Reader` [password will be auto generated]
- With the admin account still connected in Azure Data Studio, run the following query (where 'UserPrincipalName' is found on the Entra ID menu for `DB_Reader`):
```
CREATE USER [UserPrincipalName] FROM EXTERNAL PROVIDER;
ALTER ROLE db_datareader ADD MEMBER [UserPrincipalName];
```
- Disconnect from SQL server once again and edit the connection to be using Entra ID for `DB_Reader` [the login prompt through a web browser will require setting a new password for the user]
- Connect to the SQL server with this user and confirm SQL queries for reading data work [note: must select new query after logging in with a new user, cannot just run a query in a tab that was left open previously]
- Run a SQL query that attempts to drop a table and confirm an error citing no permission is returned
