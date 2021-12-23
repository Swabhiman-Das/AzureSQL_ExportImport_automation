###### Here is an automation flow using powershell that helps you to schedule and automate your regular Export import operation Using Azure Runbook.The flow works as below  

**Scaling the source database to a higer Service tier (user choise).
 
**Exporting the Database to Azure Storage account (user choise).
 
**Deleting the existing Database in the target server (user choise).

**Importing a new database with same name in the target server.
 
**STEPS TO SCHEDULE AUTOMATION IN AZURE RUNBOOK** 
 
 1) Create automation account (with run as an account YES). Import SQLserver, Azaccouts modules from the Gallery
 2) Create credentials for source and target server name it as Sourcesever and Targetserver. Username should be the SQL admin username and Password 
 3) Create the storage account and a container. copy the container URL and save in a notepad.
 4) Generate the SAS key with Full permission for the Container and save it in a notepad.  
 5) Go back to Automation account and create a runbook with PowerShell. -> Click Edit.
 6) Copy paste the Code attached and give the Variable declaration manually. 
 7) Click on test pane and Start. 
 8) On successful test , create a schedule and attach that to the runbook , which will trigger the execution automatically 

