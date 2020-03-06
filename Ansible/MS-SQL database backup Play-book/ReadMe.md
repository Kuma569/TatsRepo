## Background  
Currently Azure Stack Hub's MS-SQL HA deployment uisng **sql-2016-alwayson** ARM template is not supporting the automated backup which is otherwise supported for MS-SQL standalone database as described below.  
  
> [Automated Backup v2 for Azure Virtual Machines](https://docs.microsoft.com/en-us/azure/virtual-machines/windows/sql/virtual-machines-windows-sql-automated-backup-v2)  
  
This instruction works only if the SQL database service has been built using the **standalone** deployment.  

According to the Microsoft support and document, this is by design:  
  
> [Backup & Restore Option for SQL server](https://docs.microsoft.com/en-us/azure/virtual-machines/windows/sql/virtual-machines-windows-sql-backup-recovery#decision-matrix)   
> ![Decision Matrix](imgs/azsh-decision-matrix.png)  
  
<br>  
  
To resolve this situation, we have two options:  
   A. Use third party backup tool, such as NetBackup agent.  
   B. Use a PowerShell script and run it using Ansible.  

In this Wiki article, option B is described.  
  
<br>  
  
## Prerequisites  
Prior to this instruction, the following instructions have to be done: 
- [Ansible installation on Linux VM](https://github.dxc.com/AdvSol/AzureStackHUB-Integrations/wiki/Ansible--:-Install-&-Setup-Ansible-for-Linux-VMs#ansible-installation-on-linux-vm-ubuntu-1804)  
  -> Setting up your Ansible server. If you have it already, not necessary.
  
- [Ansible : Setup Windows Remote Manager for Ansible Automation](https://github.dxc.com/AdvSol/AzureStackHUB-Integrations/wiki/Ansible-:-Setup-Windows-Remote-Manager-for-Ansible-Automation)  
  -> For the target SQL HA server hosts. Ansible will use WinRM to connect Windows servers.      
  
<br>  
    
## Solution summary
We will run though these steps below:  
1. Create a blob storage in Azure Stack Hub.
2. Add Ansible service account to MS-SQL server as sysadmin.
3. Setup an SQL credential and PowerSell SQLHA database backup script.
4. Setup Ansible hosts and a yml playbook for SQL HA database backup.  
  
<br>  
    
### 1. Create a blob storage account in Azure Stack Hub  
1.1 Create a storage account  
- Login to the Azure Stack Hub's [user portal](https://portal.kam1.cloud1.gov.bc.ca/), and click **"+ Create a resource"** icon and select **"Storage account - blob, file, table, queue"**.  
- Fill up the necessary information and click **Create** to create an account. In this example, the following information had been used:
![Creating a storage account](https://github.dxc.com/tmorikawa/images/blob/master/integration-AzureStackHub/azsh-creating-storage-account.png)  
  
1.2 Create a blob container  
- Navigate to **Storage accouts > mssql2016hadevautobackup > Blob**, and click **"+ Container"**.  
- Fill up the information and click OK to create. In this example, the following information had been used:  
![Creating a blob container](https://github.dxc.com/tmorikawa/images/blob/master/integration-AzureStackHub/azsh-creating-blob-container.png)  
  
1.3 Take a note of storage account access key  
- Navigate to **Storage accounts > mssql2016hadevautobackup > Access keys**, and get your storage accout's **Key**:  
![Storage Account Key](https://github.dxc.com/tmorikawa/images/blob/master/integration-AzureStackHub/azsh-storage-accout-key.png)  
  Store the key in somewhere safe.


1.4 take a note of Primary blob service endpoint  
- Navigate to **Storage accounts > mssql2016hadevautobackup > Properties** , and get your storage accout's endpoint:  
![Storage Account Endpoint](https://github.dxc.com/tmorikawa/images/blob/master/integration-AzureStackHub/azsh-storage-accout-endpoint.png)  
  
<br>
  
### 2. Add Ansible service account to MS-SQL server as sysadmin
Login to the Windows machine and launch up MS SQL Server Management Studio:  
   a. Navigate to**Security > Logins**, right click and **New login**.  
   b. In the **General** tab, Enter the service account name that had been created in the previous step as a **Login name**.  
   
   c. Set **sysadmin** in the **Server Roles** tab.  
   ![MSSMS-server-roles](https://github.dxc.com/tmorikawa/images/blob/master/Integration-Ansible/mssms-server-roles.png)
 
   d. Check **Connect SQL for sa** is Granted in the **Securables** tab.  
   ![MSSMS-securables](https://github.dxc.com/tmorikawa/images/blob/master/Integration-Ansible/mssms-securables.png)


   e. Click OK to close, and log off from MSSMS.  
  
   [Note]  
    If you don't like to give a database sa permission to the ansible user, modify the permission as required. (This particular user should be visible to the all databases under this SQL host server as it will search and backup them all. )  
  
  
  
<br>  
  
### 3. Setup an SQL credential and PowerSell SQLHA database backup script  
3.1 Create a credential
On the MS-SQL HA server hosts, Run the script below:  
   ```powershell

   #==========
   #Create cred:
   #==========
   # load the sqlps module
   import-module sqlps 

   #set parameters
   $sqlPath = "sqlserver:\sql\$($env:COMPUTERNAME)"
   $storageAccount = "<your storage account name>" 
   $storageKey = "<your storage account key>" 
   $secureString = ConvertTo-SecureString $storageKey -AsPlainText -Force 
   $credentialName = "BackupCredential-$(Get-Random)"

   Write-Host "Generate credential: " $credentialName

   #cd to sql server and get instances 
   cd $sqlPath
   $instances = Get-ChildItem

   #loop through instances and create a SQL credential, output any errors
   foreach ($instance in $instances) {
   try {
        $path = "$($sqlPath)\$($instance.DisplayName)\credentials"
        New-SqlCredential -Name $credentialName -Identity $storageAccount -Secret $secureString -Path $path -ea Stop | Out-Null
        Write-Host "...generated credential $($path)\$($credentialName)." }
   catch { Write-Host $_.Exception.Message } }

   ```  
This script will generate an sql backup credential, like below:
![Backup Credential](https://github.dxc.com/tmorikawa/images/blob/master/integration-AzureStackHub/azsh-backup-credential.png)  
    
Keep this **"BackupCredential-xxxx"** value somewhere.  
  
<br>
    
3.2 Setup sql database backup script
On the MS-SQL HA server hosts, Run the script below:
   ```powershell

    #==========
   #Run backup
   #==========
   import-module sqlps

   $sqlPath = "sqlserver:\sql\$($env:COMPUTERNAME)"
   $storageAccount = "<your storage accout name>" 
   $blobContainer = "<your container name>" 
   $backupUrlContainer = "https://<primary endpoint URL>/$blobContainer/" 
   $credentialName = "BackupCredential-<backup cred number>"

   Write-Host "Backup database: " $backupUrlContainer

   cd $sqlPath
   $instances = Get-ChildItem

   #loop through instances and backup all databases (excluding tempdb and model)
   foreach ($instance in $instances) {
   $path = "$($sqlPath)\$($instance.DisplayName)\databases"
   $databases = Get-ChildItem -Force -Path $path | Where-object {$_.name -ne "tempdb" -and $_.name -ne "model"}

   foreach ($database in $databases) {
       try {
           $databasePath = "$($path)\$($database.Name)"
           Write-Host "...starting backup: " $databasePath
           Backup-SqlDatabase -Database $database.Name -Path $path -BackupContainer $backupUrlContainer -SqlCredential $credentialName -Compression On
           Write-Host "...backup complete." }
       catch { Write-Host $_.Exception.Message } } }

   ```  
   Save it as **"c:\scripts\backupDatabases_mssql2016hdautobackup.ps1"**  
  
This script will backup all databases except "tempdb" and "model" databsaes.  
    
After running the script, check if all sql databases have been saved to the blob storage:  
![Manually saved databases in the blob container ](https://github.dxc.com/tmorikawa/images/blob/master/integration-AzureStackHub/azsh-backup-databases.png)  
  
<br>

### 4.Setup Ansible hosts and a yml playbook for MS-SQL HA database backup  
4.1 On Ansible server, Edit Ansible hosts inventory file for your MS SQL HA servers.  
  
   ```bash

   sudo vi  /etc/ansible/hosts

   ```  
  
And add host group  and variables like below:  
   ```bash

   [winhadev]
   <your windows server FQDN or IP 1>  
   <your windows server FQDN or IP 2>
   .
   .
   [winhadev:vars]
   ansible_user=<Service account name>
   ansible_password=<Service account password>
   snaible_winrm_port=5986
   ansible_connection=winrm
   ansible_winrm_transport=credssp
   ansible_winrm_server_cert_validation=ignore
   #If the target server is Windows 2008, un-comment below:
   #ansible_winrm_credssp_disable_tlsv1_2=True

   ```
   **wq!** to save.  
 
[Note]  
-ansible_user & ansible_password are the ones that you have created in the [Ansible installation on Linux VM.](https://github.dxc.com/AdvSol/AzureStackHUB-Integrations/wiki/Ansible--:-Install-&-Setup-Ansible-for-Linux-VMs#ansible-installation-on-linux-vm-ubuntu-1804)  
  
<br>
    
4.2 Create an ansilbe play-book for MS-SQL HA database backup script.  
In your home directory on the ansible server, create a playbook:  
```bash

sudo vi <sqlhaDevAutoBackup.yml - your playbookname>

```  
...and add these lines:  
```yml

- name: Powershell sqldatabase backup for MS-SQL HA dev
  hosts: winhadev
  gather_facts: false

  vars:
   ansible_user: "<service account name>"
   ansible_password: "<password>"

  tasks:
   - win_command: powershell.exe
     args:
      stdin: c:\scripts\backupDatabases_mssql2016hdautobackup.ps1

```  
**wq!** to save.  
   
[Note]  
-**stdin** value would be a full path of your backup script that you have created at the step 2.2.  
  
<br>
  
4.3 Run the play-book.
```bash

sudo ansible-playbook ~/sqlhaDevAutoBackup.yml -vvvv

```  
**"-vvvv"** is a debug mode switch.  
  
You should see something like this output example:
```python

.
.
.
changed: [sql-ywgl0.kam1.cloudapp.cloud1.gov.bc.ca] => {
    "changed": true,
    "cmd": "powershell.exe",
    "delta": "0:00:04.486775",
    "end": "2020-01-30 11:15:08.383613",
    "rc": 0,
    "start": "2020-01-30 11:15:03.896837",
    "stderr": "",
    "stderr_lines": [],
    "stdout": "Windows PowerShell \r\nCopyright (C) 2016 Microsoft Corporation. All rights reserved.\r\n\r\nPS C:\\Users\\ansible01admin> c:\\scripts\\backupDatabases_mssql2016hdautobackup2.ps1\r\nBackup database:  https://mssql2016hdautobackup2.blob.kam1.cloud1.gov.bc.ca/backupcontainer/\n...starting backup:  sqlserver:\\sql\\SQL-YWGL0\\DEFAULT\\databases\\AutoHa-sample-dev\n...backup complete.\n...starting backup:  sqlserver:\\sql\\SQL-YWGL0\\DEFAULT\\databases\\master\n...backup complete.\n...starting backup:  sqlserver:\\sql\\SQL-YWGL0\\DEFAULT\\databases\\msdb\n...backup complete.\nPS SQLSERVER:\\sql\\SQL-YWGL0> ",
    "stdout_lines": [
        "Windows PowerShell ",
        "Copyright (C) 2016 Microsoft Corporation. All rights reserved.",
        "",
        "PS C:\\Users\\ansible01admin> c:\\scripts\\backupDatabases_mssql2016hdautobackup2.ps1",
        "Backup database:  https://mssql2016hdautobackup2.blob.kam1.cloud1.gov.bc.ca/backupcontainer/",
        "...starting backup:  sqlserver:\\sql\\SQL-YWGL0\\DEFAULT\\databases\\AutoHa-sample-dev",
        "...backup complete.",
        "...starting backup:  sqlserver:\\sql\\SQL-YWGL0\\DEFAULT\\databases\\master",
        "...backup complete.",
        "...starting backup:  sqlserver:\\sql\\SQL-YWGL0\\DEFAULT\\databases\\msdb",
        "...backup complete.",
        "PS SQLSERVER:\\sql\\SQL-YWGL0> "
    ]
}
META: ran handlers
META: ran handlers

PLAY RECAP ************************************************************************************************************************
sql-ywgl0.kam1.cloudapp.cloud1.gov.bc.ca : ok=1    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
.
.

```  
[Note]  
- You can check what databases were backup to where in this output.
- In the **PLAY RECAP **, **ok=1** should appear if the run was successful.
- Also check if databases are existed in the blob container on the Azure Stack Hub.  
  
<br>
  
4.4 Set up a cron job  
Add an entry to Ansible user's crontab.  
```bash

sudo crontab -u <ansible service account> -e

```  
  
The below entry will backup sql databases in every 6 hours:  
```bash

#Run ansible playbook under the user "ansible-winrm user".
0 */6 * * * sudo ansible-playbook /home/<ansible service account name>/sqlhaDevAutoBackup.yml

```  
"wq!" to save the crontab.  
  
Wait and see if this schedule yml will backup as expected.  
  
--END