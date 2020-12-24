# Table of Contents
- [Get the Powershell version](#get-the-powershell-version)
- [Get aliases](#get-aliases)
- [Get the installed modules](#get-the-installed-modules)
- [Using "more" to display long list](#using--more--to-display-long-list)
- [Re-run a previous command from history](#re-run-a-previous-command-from-history)
- [Clear history](#clear-history)
- [Capture everything that happens in the command window](#capture-everything-that-happens-in-the-command-window)
- [Install a new module](#install-a-new-module)
- [Import a module](#import-a-module)
- [Get imported modules](#get-imported-modules)
- [Save a module to disk to install it on another machine behind firewall](#save-a-module-to-disk-to-install-it-on-another-machine-behind-firewall)
- [Uninstall a module](#uninstall-a-module)
- [Search available commands](#search-available-commands)
- [Get number of commands available in a module](#get-number-of-commands-available-in-a-module)
- [Get online help on a specific command](#get-online-help-on-a-specific-command)
- [Update the help files locally](#update-the-help-files-locally)
- [Get the list of databases on an instance using dbatools](#get-the-list-of-databases-on-an-instance-using-dbatools)
- [Get services running on the local machine](#get-services-running-on-the-local-machine)
- [Get methods and properties of the powershell object](#get-methods-and-properties-of-the-powershell-object)
- [Write output to file](#write-output-to-file)
- [Display output formatted as table with specific columns and best fit](#display-output-formatted-as-table-with-specific-columns-and-best-fit)
- [Filter list for specific name](#filter-list-for-specific-name)
- [Start a specific service](#start-a-specific-service)
- [Creating a credential and using it to connect to a SQL Server](#creating-a-credential-and-using-it-to-connect-to-a-sql-server)
- [Get role members](#get-role-members)
- [Get instances on the hosts which names are stored in a text file or array](#get-instances-on-the-hosts-which-names-are-stored-in-a-text-file-or-array)
- [Copy a whole database using dbatools](#copy-a-whole-database-using-dbatools)
- [Create a new login on multiple instances using dbatools](#create-a-new-login-on-multiple-instances-using-dbatools)
- [Copy login from one server to another using Powershell](#copy-login-from-one-server-to-another-using-powershell)
- [Manipulate the SQL Server using the sqlps module](#manipulate-the-sql-server-using-the-sqlps-module)
- [Powershell script to check if a domain user exists](#powershell-script-to-check-if-a-domain-user-exists)




# Get the Powershell version
```powershell
$PSVersionTable
```

# Get aliases
```powershell
Get-Alias
```

# Get the installed modules
```powershell
Get-InstalledModule
```

# Using "more" to display long list
```powershell
Get-Command -Verb Get -Noun *DNS* | more
```

# Re-run a previous command from history
```powershell
Get-History

  Id     Duration CommandLine
  --     -------- -----------
....
   5        0.497 Get-Help -Name *DNS*
....

Invoke-History 5
```

# Clear history
```powershell
Clear-History
```

# Capture everything that happens in the command window
```powershell
Start-Transcript -Path c:\temp\transcript.txt -Append

# To stop
Stop-Transcript
```

# Install a new module
```powershell
Install-Module -Name NtpTime
```

# Import a module
```powershell
Import-Module -Name dbatools
```

# Get imported modules
```powershell
Get-Module
```

# Save a module to disk to install it on another machine behind firewall
```powershell
Save-Module -Name dbatools -Path "C:\Temp"
```

# Uninstall a module
```powershell
Uninstall-Module dbatools
```

# Search available commands
```powershell

# globally
Get-Command -Verb Get -Noun *DNS*

# in a module
Get-Command -Module dbatools

# filter the commands
Get-Command -Module dbatools | ? name -Like "*instance*"
```

# Get number of commands available in a module
```powershell
(Get-Command -Module dbatools).count

# quick statistics min, max, count ....
Get-Command -Command-Type Function | Measure-Object
```

# Get online help on a specific command
```powershell
# help online
Get-Help Get-DbaDatabase -Online

# help: alias of Get-Help, but adds "more"
help Get-Help

# display only the examples
help Get-Service -Examples

# give the full documentation
help Get-Service -Full

# man is also the alias
man Get-Service

# get the about files with the details
```

# Update the help files locally
```powershell
Update-Help
```


# Get the list of databases on an instance using dbatools
```powershell
Get-DbaDatabase -SqlInstance "<host name>\<instance name>" -ExcludeSystem
```

# Get services running on the local machine
```powershell
Get-Service
```

# Get methods and properties of the powershell object
```powershell
Get-Service | Get-Member
```

# Write output to file
```powershell
Get-Process | Out-File c:\temp\xx.txt

# or
Get-History | Out-File ....
```

# Display output formatted as table with specific columns and best fit
```powershell
Get-Service | Format-Table Name, MachineName, CanStop -AutoSize
```

# Filter list for specific name
```powershell
Get-Service | ?{$_.Name -eq "SQLWriter"}
# which is a short form of: Get-Service | Where-Object{$_.Name -eq "SQLWriter"}

# get the service on another host
Get-Service -ComputerName "<host name>" -Name "SQL Server (MSSQLSERVER)"
```

# Start a specific service
```powershell
Get-Service | ?{$_.Name -eq "SQLWriter"} | %{$_.Start()}
# which is a short form of: Get-Service | Where-Object{$_.Name -eq "SQLWriter"} | Foreach{$_.Start()}
```

# Creating a credential and using it to connect to a SQL Server
```powershell
$username = "andras"
$password = Read-Host "Please enter password for $username" -AsSecureString
$cred = New-Object System.Management.Automation.PSCredential $username, $password

$instance = Connect-DbaInstance -SqlInstance '<host name>\<instance name>' -SqlCredential $cred

# list the databases
$instance | Get-DbaDatabase
```

# Get role members
```powershell
Get-DbaServerRoleMember -SqlInstance '<host name>\<instance name>'

# get the sysadmins only
Get-DbaServerRoleMember -SqlInstance '<host name>\<instance name>' | Where-Object Role -EQ 'sysadmin'

# get the names only
$(Get-DbaServerRoleMember -SqlInstance '<host name>\<instance name>' | Where-Object Role -EQ 'sysadmin').name

# or
Get-DbaServerRoleMember -SqlInstance '<host name>\<instance name>' | Where-Object Role -EQ 'sysadmin' | Select-Object name
```


# Get instances on the hosts which names are stored in a text file or array
```powershell
$comp = Get-Content C:\Temp\instances.txt
foreach($computer in $comp){
  Find-DbaInstance -ComputerName $computer
 }


$comp2 = @('<host name 1>', '<host name 2>')
foreach($computer in $comp2){
  Find-DbaInstance -ComputerName $computer
 }
 
# the short format
 @('<host name 1>', '<host name 2>') | % {Find-DbaInstance -ComputerName $_}
```

# Copy a whole database using dbatools
```powershell
Copy-DbaDatabase -Source '<host name>\<instance name>' -Database '<database name>' -Destination '<host name>\<instance name>' -BackupRestore -SharedPath '<path visible to both servers>'  -Force
```

# Create a new login on multiple instances using dbatools
```powershell
New-DbaLogin -SqlInstance '<host name>\<instance name 1>','<host name>\<instance name 2>'  -ServerRole sysadmin  -Login '<login>' -Force'

# granting access to a database

$user = New-DbaDBUser -SqlInstance '<host name>\<instance name>' -Database '<database name>' -Login '<login>'
$user.AddToRole('db_datareader')
```

# Copy login from one server to another using Powershell
```Powershell
Copy-DbaLogin -Source <source server> -Destination <destination server>
```

# Manipulate the SQL Server using the sqlps module
```powershell

# import module
Import-Module sqlps -Verbose
psdrive

cd SQLSERVER
dir

cd SQLSERVER:\SQL\<host name>\<instance name>\Databases\<database name>\Tables

# get the name and row count of each table
SQLSERVER:\SQL\<host name>\<instance name>\Databases\<database name>\Tables>dir | Format-Table Name, RowCount

# get the backup data of each database
SQLSERVER:\SQL\<host name>\<instance name>\Databases>dir | Format-Table Name, LastBackupDate, LastDifferentialBackupDate, LastLogBackupDate -AutoSize

# get the parameters of a stored procedure
SQLSERVER:\SQL\<host name>\<instance name>\Databases\<database name>\StoredProcedures\<stored procedure name with schema>\Parameters> dir
```

# Powershell script to check if a domain user exists
You will need to download the Quest_ActiveRolesManagementShellforActiveDirectoryx64_151.zip file from https://www.powershelladmin.com/wiki/Quest_ActiveRoles_Management_Shell_Download
```powershell
Add-PSSnapin Quest.ActiveRoles.ADManagement -ErrorAction Stop

$username = "<domain>\<user>"

if (Get-QADUser -Identity $username)
{Write-Host "It's alive"}
else
{Write-Host "Account does not exist."}
```
