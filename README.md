## Automate Azure SQL DB Export & import using Azure Runbook.
###### *****Flow of Code*****

**Scaling the source database to a higer Service tier (user choice).
 
**Exporting the Database to Azure Storage account (user choice).
 
**Deleting the existing Database in the target server (user choice).

**Importing a new database with same name in the target server.
 
**STEPS TO SCHEDULE AUTOMATION IN AZURE RUNBOOK** 
 
 1) Create automation account (with run as an account YES). Import SQLserver, Azaccouts modules from the Gallery
 2) Create credentials for both source and target server and save it as Sourcesever and Targetserver. Username should be the SQL admin username and Password 
 3) Create the storage account and a container. copy the container URL and save in a notepad.
 4) Generate the SAS key with Full permission for the Container and save it in a notepad.  
 5) Go back to Automation account and create a runbook with PowerShell. -> Click Edit.
 6) Copy paste the Code attached and give the Variable declaration manually. 
 7) Click on test pane and Start. 
 8) On successful test , create a schedule and attach that to the runbook , which will trigger the execution automatically 
 9) Detiled Scheduling Information can be found in the "SETUP_INDETAILED" Doc

```

####################### Varrible Declaration ####################################
$sourceresourcegroup='myrg'
$sourceServer = 'mymssqlserver11'
$sourceDB = 'testcmk' 
$ScaleSourceDBedition= 'Standard'
$scalesourceDBSLO = 'S0'
$TargetResourcegroup = 'myrg'
$targetServer='testsouth'
$targetDB='target'
$targetEdition = 'standard'
$targetslo = 'S2'
$targetmaxsizeofDB = 2147483648
$BaseStorageUri = "https://newaudit.blob.core.windows.net/exportdb"
$bacpacFilename = "$sourceDB" + (Get-Date).ToString("yyyy-MM-dd-HH-mm") + ".bacpac"
$BacpacUri = $BaseStorageUri + "/test/" + $bacpacFilename
$StorageKeytype = "SharedAccessKey"
$StorageKey = "sp=racwdl&st=2022-01-16T08:03:47Z&se=2022-08-11T16:03:47Z&sv=2020-08-04&sr=c&sig=nDSNoz%2FPKnwyCFCQULd1r1PCmBn0CEqnuwh5Yt1hrYk%3D"
$SourceCred = Get-AutomationPSCredential -Name "SourceServer"
$TargetCred = Get-AutomationPSCredential -Name "TargetServer"


#Establish a connection to ARM through AZ 

$connectionName = "AzureRunAsConnection"
try
{
    # Get the connection "AzureRunAsConnection "
    $Conn = Get-AutomationConnection -Name $connectionName   
    "Logging in to Azure..."
    Connect-AzAccount -ServicePrincipal -Tenant $Conn.TenantID -ApplicationId $Conn.ApplicationID -CertificateThumbprint $Conn.CertificateThumbprint
    }
catch {
    if (!$Conn)
    {
        $ErrorMessage = "Connection $connectionName not found."
        throw $ErrorMessage
    } else{
        Write-Error -Message $_.Exception
        throw $_.Exception
    }
}




###################  Function Defination  ############################################

function Checkimportexport($operationID)
{
$Actualstatus = Get-AzSqlDatabaseImportExportStatus -OperationStatusLink $operationID |select -ExpandProperty  status
While( $Actualstatus -eq "InProgress")
   {
   $Actualstatus = Get-AzSqlDatabaseImportExportStatus -OperationStatusLink $operationID |select -ExpandProperty  status
   start-sleep -s 10
   if($Actualstatus -eq "Succeeded")
     {
      'import/export sucessfull'
      break
     }
    elseif($Actualstatus -eq "failed")
      {
        'impot/export Failed'
       break
      }
 }
 return $Actualstatus
}


function ScalingConfirmation($scaledDB)
{
 $ScalingStats =  Get-AzSqlDatabase -ResourceGroupName $sourceresourcegroup -ServerName $sourceServer -DatabaseName $sourceDB | select -Expandproperty CurrentServiceObjectiveName
 while($ScalingStats -ne $scalesourceDBSLO )
  {
      $ScalingStats =  Get-AzSqlDatabase -ResourceGroupName $sourceresourcegroup -ServerName $sourceServer -DatabaseName $scaledDB | select -Expandproperty CurrentServiceObjectiveName
      start-sleep -s 10
  }
  return $ScalingStats
}

function Checkdropstats($dropDbname)
  {
     $Dstatus = Get-AzSqlDatabase -ResourceGroupName $TargetResourcegroup -ServerName $targetServer -DatabaseName $dropDbname | Select -ExpandProperty DatabaseName
     while($Dstatus -eq $dropDbname)
      {
        start-sleep -s 10
        $Dstatus = Get-AzSqlDatabase -ResourceGroupName $TargetResourcegroup -ServerName $targetServer -DatabaseName $dropDbname | Select -ExpandProperty DatabaseName

      }
  return $Dstatus
  }


####################  Code flow Phase 1 - export to Storage  ########################################################## 

$currentSKU = Get-AzSqlDatabase -ResourceGroupName $sourceresourcegroup -ServerName $sourceServer -DatabaseName $sourceDB 

if($currentSKU.CurrentServiceObjectiveName -ne $scalesourceDBSLO)
{
 "start Scaling"
  $Scaling = Set-AzSqlDatabase -ResourceGroupName $sourceresourcegroup -ServerName $sourceServer -DatabaseName $sourceDB -Edition $ScaleSourceDBedition -RequestedServiceObjectiveName $scalesourceDBSLO |select -Expandproperty CurrentServiceObjectiveName
}
$scalingresult = ScalingConfirmation -scaledDB $Scaling.DatabaseName
if($scalingresult -eq $scalesourceDBSLO)
  {
      Write-Verbose -Message "scaling suceeded"
  }
$exportRequest = New-AzSqlDatabaseExport -ResourceGroupName $sourceresourcegroup -ServerName $sourceServer -DatabaseName $sourceDB  -StorageKeytype $StorageKeytype -StorageKey $StorageKey -StorageUri $BacpacUri -AuthenticationType Sql -AdministratorLogin $SourceCred.Username -AdministratorLoginPassword $SourceCred.password
$exportStatus= Checkimportexport -operationID $exportRequest.OperationStatusLink
if($exportStatus -eq 'Succeeded')
  {
      $scaleback = Set-AzSqlDatabase -ResourceGroupName $sourceresourcegroup -ServerName $sourceServer -DatabaseName $sourceDB -Edition $currentSKU.edition -RequestedServiceObjectiveName $currentSKU.CurrentServiceObjectiveName
    }
##################  Code flow phase 2 - Import to Server  ###############################################################


if($exportStatus -eq "Succeeded")
{
    $Checktarget = Get-AzSqlDatabase -ResourceGroupName $TargetResourcegroup -ServerName $targetServer -DatabaseName $targetDB | Select -ExpandProperty DatabaseName
}
   if($Checktarget -eq $targetDB)
    {
           $drop=Remove-AzSqlDatabase -ResourceGroupName $TargetResourcegroup -ServerName $targetServer -DatabaseName $targetDB  | Select -ExpandProperty DatabaseName
           $dropstatus = Checkdropstats($targetDB)
           if($dropstatus -ne $targetDB)
           {
           $importRequest = New-AzSqlDatabaseImport -ResourceGroupName $TargetResourcegroup -ServerName $targetServer -DatabaseName $targetDB -StorageKeyType $StorageKeytype -StorageKey $StorageKey -StorageUri $BacpacUri -Edition $targetEdition -ServiceObjectiveName $targetslo -DatabaseMaxSizeBytes $targetmaxsizeofDB -AuthenticationType Sql -AdministratorLogin $TargetCred.Username  -AdministratorLoginPassword $TargetCred.password
           }
        }
    else
      {
        $importRequest = New-AzSqlDatabaseImport -ResourceGroupName $TargetResourcegroup -ServerName $targetServer -DatabaseName $targetDB -StorageKeyType $StorageKeytype -StorageKey $StorageKey -StorageUri $BacpacUri -Edition $targetEdition -ServiceObjectiveName $targetslo -DatabaseMaxSizeBytes $targetmaxsizeofDB -AuthenticationType Sql -AdministratorLogin $TargetCred.Username  -AdministratorLoginPassword $targetcred.password
      }
 $importStatus = Checkimportexport -operationID  $importRequest.OperationStatusLink

```
