Are you trying to achieve something similar to Database refresh and run out of options ? Well it is well known as Azure SQL Does not allows you to take manual backup and restore featues but a sound way to achieve that is bya regular export and Import untill you really set up a ETL pipeline for your infrastructure.


Here is an automation flow using powershell that helps you to schedule and automate your regular Export import operation Using Azure Runbook.The flow works as below  

 ** Scaling the source database to a higer Service tier (user choise).
 
 ** Exporting the Database to Azure Storage account (user choise).
 
 ** Deleting the existing Database in the target server (user choise).
 
 ** Importing a new database with same name in the target server. 
