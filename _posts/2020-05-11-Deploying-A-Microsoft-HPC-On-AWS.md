---
layout: post
title: "Deploying a Microsoft High Performance Cluster on AWS using Infrastructure as Code."
date: 2020-05-16
summary: "When running a HPC on AWS you want to use CloudFormation templates to create any instance needed to run the HPC cluster. HPC needs a SQL Server, Headnode(s) and ComputeNodes, we will create template and script for all the components needed to run an High Performance Cluster on Amazon Web Services"
minute: 30
---

>Microsoft HPC Pack brings the power of high-performance computing (HPC) to the commercial mainstream. 
>The centralized management and deployment interface helps to simplify deployment for both large and small compute
>clusters and provide a simple and effective management experience to increase cluster administrator productivity.

You can find more information about HPC on the <a href ="https://docs.microsoft.com/en-us/powershell/high-performance-computing/overview-of-microsoft-hpc-pack?view=hpc16-ps">Microsoft Site <a> 

When running on Microsoft Azure, you can use the ARM templates provided by Microsoft to install an HPC. You can also use the PowerShell scripts on the internet to install HPC on Virtual Machines manually. You can log on to a VM and install everything as an administrator. Nowadays, we want to use Infrastructure as Code and use scripts and pipelines to install Infrastructure like Virtual Machines on Microsoft Azure or EC2 Instances on Amazon Web Services (AWS). This way, we can redeploy the same setup on each new environment without worrying about run-book, differences, and manual errors. 

In this blog, you will learn about deploying an HPC on AWS using CloudFormation templates (in YAML) and PowerShell scripts. I will highlight some things to remember when installing an HPC. 

# CloudFormation templates

When you want to use infrastructure as code to deploy EC2 instances on AWS, you can use CloudFormation files (in YAML). You can use it to deploy separate instances by creating a template for one single EC2 instance. You can also use an Auto Scaling Group, where you can deploy multiple instances at the same time. You can define a minimum and a maximum number of instances. AWS will make sure there is always a minimal number of instances running. We will create three Auto Scaling Groups for running our SQL Server, to run the HeadNode and the ComputeNodes.  

To install the HPC software, we will use the `user data` option in the CloudFormation templates. This will install the software when the machine is booted; we will create scripts to make sure this is only installed and configured when it is not installed yet. If the installation fails, we can just terminate the instance, and the auto scaling group will automatically start a new instance.   

Before we can install the HPC software, we need to make sure it is available on the instances. We used Packer to create `Amazon Machine Images (AMI)` and used `Chocolately` to install the AWS Command Line Interface. With this CLI, we can use run AWS instructions to copy software from an `S3 Bucket` to our AMI. 

The HPC has three components we need to install, a Microsoft SQL Server, one or multiple HeadNodes and, one or more ComputeNodes. 

# SQL Server 

The first thing we need is a running Microsoft SQL Server Instance. We used the Microsoft SQL Server AMI provided by Microsoft on AWS to create an AMI, which we will use to create our database server. Like we mentioned before, we installed AWS CLI on the AMI. Because we use an AMI with SQL Server already installed, we do not need to worry about installing the SQL Server software.  

When downloading the HPC Pack software, Microsoft provides Powershell scripts to create and install the database on the SQL Server instance. But the instructions require you to log in to the machine and run the scripts as an administrator using Powershell. We changed the scripts for creating the database a little. We want to create the database just once on the first time we create the SQL Server, the next time we create a SQL instance we want to reuse the database and not restart all over again. We will create the database on the D-drive and use a different volume. You can reuse the volumes on AWS instances, find out how by using <a href="https://arjanvanbekkum.github.io/blog/2020/04/29/Reconnect-Volumes-On-AWS-EC2-instances">this</a> blog I wrote  

### Create the databases

Before you can create a database in a folder on a drive, you need to make sure that the SQL server user has permissions to create the database. First, you need to create the folder on the drive to store the database and next allow the SQL server to create the database files in the folder. We will save this script as `setupsqlinstance.ps1`.

```powershell
$path = "D:\SQL"
New-Item -ItemType Directory -Force -Path $path
$Acl = (Get-Item $path).GetAccessControl('Access')
$Ar = New-Object System.Security.AccessControl.FileSystemAccessRule("NT SERVICE\MSSQLSERVER", 
    "FullControl", "ContainerInherit,ObjectInherit", "None", "Allow")
$Acl.SetAccessRule($Ar)
Set-Acl $path $Acl
```

We need to create a SQL script for creating the databases, to run an HPC you need to create the following databases:
- HPCManagement
- HPCScheduler
- HPCReporting
- HPCDiagnostics
- HPCMonitoring

This SQL script will create the `HPCManagement` database if it does not exist, and if it does, it will reconnect the database. We will create an `SQL_CREATE_DB` statement to create the database. If the database files exist, we will add the `FOR ATTACH` statement to reuse the database and register it in SQL Server. Next, we will verify the database doesn't exist and execute the query. In the script below, you will find the SQL script to create one database; you can extend the script to create all the databases you need, the size and growth are the same as below. We will save this script as `CreateHpcDatabase.sql`

```sql
USE master
GO

DECLARE @hpc_datapath nvarchar(256);
SET @hpc_datapath = 'D:\SQL';
DECLARE @DBNAME nvarchar(256);
DECLARE @SQL_CREATE_DB nvarchar(2000);
DECLARE @exist INT

SET @DBNAME = @hpc_datapath +'\HPCManagement.mdf'
exec master.dbo.xp_fileexist @DBNAME, @exist OUTPUT
SET @exist = CAST(@exist AS BIT)

SET @SQL_CREATE_DB = 'CREATE DATABASE HPCManagement 
ON (
	NAME = HPCManagement_data, 
	FILENAME = ''' + @hpc_datapath + '\HPCManagement.mdf'', 
	size = 1024MB, 
	FILEGROWTH  = 50% 
) 
LOG ON 
( 
	NAME = HPCManagement_log,
	FILENAME = ''' + @hpc_datapath +'\HPCManagement.ldf'',
	size = 128MB,	
	FILEGROWTH  = 50% 
)'

IF (@exist = 1)
  SET @SQL_CREATE_DB = @SQL_CREATE_DB + ' FOR ATTACH;'

IF (NOT EXISTS (SELECT name FROM master.dbo.sysdatabases WHERE ([name] = 'HPCManagement' ) ) )
IF NULLIF(@hpc_datapath, '') IS NOT NULL
EXECUTE (@SQL_CREATE_DB)
```

### Setting Up Mixed Mode Authentication

So now that we have a SQL script to create the databases, we a PowerShell to run this script. To connect to our SQL server, we will use SQL server authentication. The default authentication in SQL Server is Windows authentication, so we need to change this. We can do this within SQL Server, but we need to restart the SQL server services to activate it. We created a script containing the one line below and saved as `update_sql_mixed_mode.sql`. 

```sql
EXEC xp_instance_regwrite N'HKEY_LOCAL_MACHINE', N'Software\Microsoft\MSSQLServer\MSSQLServer', 
    N'LoginMode', REG_DWORD, 2 
```

The last step for the SQL server is to connect all the pieces and create a PowerShell we can use to run in the `UserData` option in the CloudFormation template. This PowerShell will run our SQL script on the SQL server itself, so we can simply use `localhost` to connect.  

### Creating the users

One last thing before we can create our database, we need a user to for connecting to the database. Microsoft provides two scripts in the HPC pack to create database users; we just need to call them from within this PowerShell we are creating. We want to make sure that we do not store username or password in our source code repository. And you probably also want to use a different user for the different environments. We used the AWS parameter store to store the encrypted secrets, by calling the `aws ssm get-parameter` we can get the parameters from the store and use them in our script. If they change, we do not need to change any of the code because they are fetched every time we run the script. Now we can use these parameters to create the users in the databases.   

```powershell
$ServerInstance = "localhost"

Invoke-Sqlcmd -ServerInstance $ServerInstance -InputFile "C:\programdata\installdata\SQL\update_sql_mixed_mode.sql"

# stop / start sql server to activate mixed mode authentication
net stop sqlserveragent
net stop mssqlserver 

net start mssqlserver
net start sqlserveragent 

Invoke-Sqlcmd -ServerInstance $ServerInstance -InputFile "C:\programdata\installdata\SQL\CreateHpcDatabase.sql" 

$password = aws ssm get-parameter --name "/sql-server/service-account/password" 
    --query "Parameter.Value" --with-decryption --output text --region eu-central-1
$HpcUser = aws ssm get-parameter --name "/sql-server/service-account/username" 
    --query "Parameter.Value" --with-decryption --output text --region eu-central-1

$ParameterArray = "TargetAccount=$HpcUser", "PassWord=$password"
$hpcDBs = @('HPCDiagnostics', 'HPCManagement', 'HPCMonitoring', 'HPCReporting', 'HPCScheduler')
foreach($hpcdb in $hpcDBs)
{
    Invoke-Sqlcmd -ServerInstance $ServerInstance -Database $hpcdb 
        -InputFile "C:\programdata\installdata\SQL\AddDbUserForHpcSetupUser.sql" -Variable  $ParameterArray 
    Invoke-Sqlcmd -ServerInstance $ServerInstance -Database $hpcdb 
        -InputFile "C:\programdata\installdata\SQL\AddDbUserForHpcService.sql" -Variable  $ParameterArray 
}
```

### Setting up CloudFormation template yaml

We need to make sure this PowerShell scripts run every time we boot our SQL server instance. To make that happen, we will add these scripts to the `UserData` option of the `LaunchTemplate` in the CloudFormation file. We have created a `createdisks.ps1` script to connect the disks every time we reboot our server, you can find the link to the blog earlier in this blog. Setting the `persist` tag to true will ensure this script will run every time the instance is booted.

```yaml
  LaunchTemplate:
    Type: AWS::EC2::LaunchTemplate
    Properties:
      LaunchTemplateName: !Ref AWS::StackName
      LaunchTemplateData:
        ImageId: 'ReferenceTheTheAMIFile'
        InstanceType: z1d.large
        IamInstanceProfile: 'YourProfile'
        SecurityGroupIds: 'YourSecurityGroup'
        TagSpecifications:
          - ResourceType: instance
            Tags:
              - Key: Name
                Value: SQLServer
              - Key: network-interface-manager-pool
                Value: SQLServer
        UserData:
          Fn::Base64: 
            Fn::Sub: |
              <powershell>
                Write-Host "Mounting volumes and disks"
                Invoke-Expression -Command "& 'C:\\programdata\\installdata\\createdisks.ps1' "
                Write-Host "Mounting done"
                
                Write-Host "Setup SQL Instance"
                Invoke-Expression -Command "& 'C:\\programdata\\installdata\\setupsqlinstance.ps1' "

                Write-Host "Create HPC databases"
                Invoke-Expression -Command "& 'C:\\programdata\\installdata\\setuphpcdatabase.ps1' "
              </powershell>
              <persist>true</persist>
```

So now we have a running SQL Server instance, and we are almost ready to connect the `HeadNode.` We created the user, so we should be able to connect to the instance, but we are on a cluster. If the SQL server instance is recreated, the IP-address and server name will change. So we need to add something that will not change and what we can use to always make a connection. Therefore we will add a DNS record using `AWS Route53` and connect that to the SQL server, and later on, will we create another one for the HeadNode. We also added an IP-address that never changes. 

First, you need to create a `NetworkInterface,` you can use the `network-interface-manager-pool` tag to connect this to the SQL server instance.

```yaml
  SQLServerNIC:
    Type: AWS::EC2::NetworkInterface
    Properties:
      SubnetId: 'YourSubNet'
      GroupSet: 'YourSecurityGroup'
      PrivateIpAddress: !Sub
        - ${IpAddress}
        - IpAddress: !Join
            - .
            -   - !Select [0, !Split [., !Select [0, !Ref SubnetsCidr]]]
                - !Select [1, !Split [., !Select [0, !Ref SubnetsCidr]]]
                - !Select [2, !Split [., !Select [0, !Ref SubnetsCidr]]]
                - '5'
      Tags:
        - Key: Name
          Value: SQLServer
        - Key: network-interface-manager-pool
          Value: SQLServer
```

Now we can create a `RecordSetGroup` and connect that to the `NetworkInterface` IP-address. The name of the `RecordSets` contains the DNS-name we can use to connect to our SQL Server instance.  

```yaml
   SQLServerDNSRecord:
    Type: AWS::Route53::RecordSetGroup
    Properties:
      HostedZoneId: !Ref PrivateHostedZoneId
      RecordSets:
        - Name: 'mssql.mydomain.com'
          Type: A
          TTL: '60'
          Weight: 1
          SetIdentifier: sql-server
          ResourceRecords:
            - !GetAtt SQLServerNIC.PrimaryPrivateIpAddress
```

So that it for creating the SQL Server, now let's move to the HeadNode.

# HeadNode

>HPC Pack includes a highly scalable job scheduler that provides support for interactive Service-Oriented Architecture (SOA) 
>applications using High Performance Computing for Windows Communication Foundation (HPC for WCF) and parallel jobs using the 
>Microsoft Message Passing Interface (MS-MPI). Essential applications from key independent software providers (ISVs) can be run 
>on the cluster to help you meet your business needs in a timely, cost-effective, and highly productive manner.
 

The HeadNode makes sure jobs for the HPC are distributed correctly over the ComputeNodes. It holds the network topology and node templates you can assign to the ComputeNodes. You can find more information about the complete HPC pack on the Microsoft site, use the link at the top of this blog. 

### Create a self-signed certificate

The first thing we need to make sure when installing the HPC Pack software on the HeadNode is that the SQL Server instance is up and running. The installation of the HPC Pack verifies the connection with the SQL Server. Next, we need a certificate to communicate over HTTPS, Microsoft provides a PowerShell script called `CreateCertificate.ps1` in the installation pack of the HPC software. In this blog, we will use this certificate. As you might know, you can only export a certificate if you use a password; in the Microsoft script, there is a default password. However, this is not what we want to use, so we changed a few lines in the PowerShell but left the rest untouched.

Add the top of the script we added the following lines. First, we changed the `CommonName` into something we could recognize, and we changed the `Path` into where we wanted the certificate to be saved. Like with the SQL Server users, we added another password to the `AWS Parameter Store`. We will need this password to export the certificate, but we also need it to install the HeadNode software later on. 

```powershell
$CommonName = "HPC Pack 2016 Communication"
$Path = "C:\ProgramData\installdata\HPCCertificate.pfx"
$password = aws ssm get-parameter --name "/certificate/hpc/password" --query "Parameter.Value" --with-decryption --output text --region eu-central-1
$password = $password | ConvertTo-SecureString -AsPlainText -Force
``` 

At the bottom of the script, we changed in lines below to make sure the certificate is saved to the `Path` we need and uses the password we stored in the `AWS Parameter Store`

```powershell
PFXString = $enrollment.CreatePFX([Runtime.InteropServices.Marshal]::PtrToStringAuto([Runtime.InteropServices.Marshal]::SecureStringToBSTR($Password)), 0)
Set-Content -Path $Path -Value ([Convert]::FromBase64String($PFXString)) -Encoding Byte
# Remove the certificate from Cert:\CurrentUser\My\$thumbprint store
Remove-Item Cert:\CurrentUser\My\$thumbprint -Confirm:$false -Force
```

### Installing the HPC Pack on the HeadNode.

When we created the AMI for installing the HeadNode, we used the AWS CLI to copy the HPC Pack software from an `AWS S3 Bucket` to the AMI. We will add this script to the `UserData` of the EC2 instance. These scripts only need to install the software when it is not running already. At the top of the script, we add a check to verify that the schedular service is installed. 

```powershell
# if already installed using the hpc scheduler service installed in this script
if (Get-Service HpcScheduler -ErrorAction SilentlyContinue)
{
    Write-Host "HPC Pack already installed, skipping..."
}
```

So if this service isn't installed, we can install the HPC Pack on the HeadNode. The HPC Pack installer also provides a command-line interface. You can provide the parameters you need to install the HPC Pack unattended. Before we can run the installer, we will create the command line arguments in a single character string and pass this as an `argumentlist` to the installer. 

We need both the information about the certificate as well as the SQL Server and the SQL-User for connecting to the database. We will get them from the `AWS Parameter Store` and store them in variables we later on need. 

```powershell
$certificate_password = aws ssm get-parameter --name "/certificate/hpc/password" --query "Parameter.Value" --with-decryption 
    --output text --region eu-central-1
$sql_password = aws ssm get-parameter --name "/sql-server/service-account/password" --query "Parameter.Value" --with-decryption 
    --output text --region eu-central-1
$sql_user = aws ssm get-parameter --name "/sql-server/service-account/username" --query "Parameter.Value" --with-decryption 
    --output text --region eu-central-1

$tgtdir = "C:\ProgramData\installdata\HPC\2016\"
$certpath = "C:\ProgramData\installdata\HPCCertificate.pfx"

$ClusterName = "HeadNode.mydomain.com"
$SQLServerInstance = "mssql.mydomain.com"
```

Let's pass this information into one big `ArgumentList`. We need to tell the HPC Pack software we want to install the HeadNode version of the software, and we want to run without any user interaction, so the first two parameters are `-unattend` and `-HeadNode`. We will add a name to our cluster using the DNS HeadNode entry we will create in our CloudFormation file. For the installation, we need the just created certificate and the password we used to export the certificate to disk. 

```powershell
$setupArg = "-unattend -HeadNode -ClusterName:$ClusterName -SSLPfxFilePath:$certpath -SSLPfxFilePassword:$certificate_password"
```

It is time to create a connection to the database on the running SQL Server instance. We need to specify a connection string to each database separately. We know they all exist on the same server, so we can reuse the connection to the server with the `sql_user` and `sql_password` variables. Next, we can specify the connection string to each of the databases and add them all to the `setupArg` variable. The HPC Pack installer will create tables and views in the database using this connection string. 

```powershell
$secinfo = "Integrated Security=False;User ID=$sql_user;Password=$sql_password"
$mgmtConstr = "Data Source=$SQLServerInstance;Initial Catalog=HpcManagement;$secinfo"
$schdConstr = "Data Source=$SQLServerInstance;Initial Catalog=HpcScheduler;$secinfo"
$monConstr  = "Data Source=$SQLServerInstance;Initial Catalog=HPCMonitoring;$secinfo"
$rptConstr  = "Data Source=$SQLServerInstance;Initial Catalog=HPCReporting;$secinfo"
$diagConstr = "Data Source=$SQLServerInstance;Initial Catalog=HPCDiagnostics;$secinfo"
$setupArg = "$setupArg -MGMTDBCONSTR:`"$mgmtConstr`" -SCHDDBCONSTR:`"$schdConstr`" -RPTDBCONSTR:`"$rptConstr`" 
    -DIAGDBCONSTR:`"$diagConstr`" -MONDBCONSTR:`"$monConstr`""           
```

The last thing we need to do is installing the HPC software on the EC2 instance by calling the installer and passing the `setupArg` as the argument list. We will wait until the process finishes using the `-Wait` option from the `Start-Process` command in PowerShell. Installing the HeadNode software may take a while. It also might fail, you can take a look at the log files in the `Windows\Temp` folder, this is where the HPC Pack creates a new folder every time the installer is started. So make sure you are able to logon to the server or can connect using remote PowerShell, the AWS Console offers options to connect using a command-line interface. 

```powershell
$p = Start-Process -FilePath "$tgtdir\setup.exe" -ArgumentList $setupArg -PassThru -Wait
if($p.ExitCode -eq 0)
{
    LogWrite "Succeed to Install HPC Pack HeadNode"
    break
}
if($p.ExitCode -eq 3010)
{
    LogWrite "Succeed to Install HPC Pack HeadNode, a reboot is required."
    break
}
```    

### Configure the HeadNode

After the installation of the HPC Pack software, we need to configure the HeadNode. The HPC pack provides us with a PowerShell module we can use to complete the configuration. After activating the snap-in, we need we can get the network interface we need. In this case, we have two, because we will create two in the CloudFormation file, the same way as we did in the SQL Server deployment. We will select the first one and set the `HpcNetwork` on `Enterprise` using this network interface.

>Enterprise network: An organizational network connected to the HeadNode and, in some cases, to other nodes in the cluster. The enterprise network is often the public or organization network that most users log on to perform their >work. All intra-cluster management and deployment traffic is carried on the enterprise network unless a private network and an optional application network also connect the cluster nodes.

```powershell
Add-PSSnapin Microsoft.HPC
$status = (Get-Service -Name HpcScheduler -ErrorAction SilentlyContinue -ErrorVariable StErr | Select-Object -ExpandProperty Status)
$NIC = Get-HpcNetworkInterface -ErrorAction Stop
Set-HpcNetwork -Topology Enterprise -Enterprise $NIC.Name[0] -EnterpriseFirewall $null
``` 

We have added the network setting to the cluster; the next step is to let is use the correct credentials when starting jobs. Again we will get the information we need from the `AWS Parameter Store`. We will also create a `PSCredential` using the username and password. We will use the `PSCredential` to pass on to the HPC cluster configuration methods for the `HpcJobCredentials` and `HpcClusterProperty`.  

```powershell
$username = aws ssm get-parameter --name "/service-account/username" --query "Parameter.Value" --with-decryption --output text --region eu-central-1
$password = aws ssm get-parameter --name "/service-account/password" --query "Parameter.Value" --with-decryption --output text --region eu-central-1
$password = $password | ConvertTo-SecureString -AsPlainText -Force
$credential = New-Object System.Management.Automation.PSCredential($username,$password)

Set-HpcJobCredential -Credential $credential
Set-HpcClusterProperty -InstallCredential $credential
```

The PowerShell plugin offers a lot of options for the configuration of the HPC Pack. We will not discuss them all here, but what you might need to do is add users to the HPC Pack using the `Add-HpcMember` and provide a `-Role` parameter, you can use `Administrator`, `JobAdministrator` or `User`

The last step is setting up the HeadNode and bringing it `online`. The HeadNode needs to be online; else it will not accept any ComputeNode connections.

add users

```powershell
Add-HpcMember -Name $item -Role administrator -ErrorVariable Err -ErrorAction Stop -WarningAction Stop
Set-HpcNode -Name $env:COMPUTERNAME -Role BrokerNode
Set-HpcNodeState -Name $env:COMPUTERNAME -State online
```

### Share the certificate

So we are almost done with setting up the HeadNode. There is only one thing we need to do, and that is, make the certificate available for the ComputeNodes. Because we are using a self-signed certificate, the ComputeNodes need to install the same certificate so we can use a trusted HTTPS connection between the nodes. This leaves us with a problem, because we are installing the HeadNode and the certificate is in a folder on our C-Drive. The solution is not that complicated; during the installation of the HPC Software a share is created, which can be used by the ComputeNodes during setup. So we copy the certificate we used to this shared folder (`C:\Program Files\Microsoft HPC Pack 2016\Data\InstallShare\Certificates\`). 

### Setting up CloudFormation template yaml 

The HeadNode CloudFormation files are almost the same as the once we used to deploy the SQL Server instance. We used Packer to create a new AMI based on the standard Windows Server version provided by Microsoft. We added `Chocolately` and the `AWS CLI`. For running the HeadNode we created an Auto Scaling Group with a launch template and provided the HeadNode with a DNS name called `HeadNode.mydomain.com`. The only thing different in the `LaunchTemplate` is, of course, the user-data. 

First, we install the certificate by calling the little modified PowerShell script provided by Microsoft. Next, we will install the HPC Pack using credentials from the parameter store, and the last step is we copy the certificate we use to install the HeadNode to the share created by the installer.

```yaml
UserData:
  Fn::Base64: 
    Fn::Sub: |
      <powershell>
        $username = aws ssm get-parameter --name "/service-account/username" --query "Parameter.Value" --with-decryption --output text --region eu-central-1
        $password = aws ssm get-parameter --name "service-account/password" --query "Parameter.Value" --with-decryption --output text --region eu-central-1
        $password = $password | ConvertTo-SecureString -AsPlainText -Force
        $credential = New-Object System.Management.Automation.PSCredential($username,$password)

        Write-Host "Create HPC Certificate"
        Invoke-Expression -Command "& 'C:\\programdata\\installdata\\CreateHpcCertificate.ps1' "
        Write-Host "Done Create HPC Certificate"

        Write-Host "Installing HPC Pack"
        Invoke-Command -ComputerName . -Credential $credential -File C:\\programdata\\installdata\\install_hpc_HeadNode.ps1
        Write-Host "Done Installing HPC Pack"

        $certpath = "C:\ProgramData\installdata\HPCCertificate.pfx"
        Copy-Item -Path $certpath -Destination 'C:\Program Files\Microsoft HPC Pack 2016\Data\InstallShare\Certificates\'
      </powershell>
      <persist>true</persist>
```

# ComputeNde

The ComputeNode will run the jobs provided by the HeadNode based on the template which is configured on the ComputeNode.

### Copy the certificates

Before we can install the software on the ComputeNode we need to copy the certificates we need from the share on the HeadNode. Except for the certificate we copied in there, there is another certificate that we must use to communicate with the HeadNode. The first step is to copy them both from the share to a local folder on the ComputeNode.

We are running a deployment from an `AWS CodeBuild` pipeline, and therefore we do not have permission to connect to the HeadNode without the correct credentials. Remember, we installed the HPC Pack on the HeadNode using specific credentials, we now use those credentials again to connect to the share with the certificates. We use these credentials to create a drive using the `New-PSDrive` command to the share on the HeadNode and use this drive to copy the certificates from the HeadNode to the ComputeNode.

```powershell
$username = aws ssm get-parameter --name "/service-account/username" --query "Parameter.Value" --with-decryption --output text --region eu-central-1
$password = aws ssm get-parameter --name "/service-account/password" --query "Parameter.Value" --with-decryption --output text --region eu-central-1
$password = $password | ConvertTo-SecureString -AsPlainText -Force
$credential = New-Object System.Management.Automation.PSCredential($username,$password)

New-PSDrive -Name "InstallDir" -PSProvider "FileSystem" -Root "\\HeadNode.mydomain.com\REMINST" -Credential $credential
Copy-Item InstallDir:\certificates\*.* -Destination "C:\ProgramData\QRM"
```

### Installing the HPC Pack on the ComputeNode.

To install the software on the ComputeNode you can use the same installer as we used on the HeadNode. The difference is that we need to tell the installer is needs to install the ComputeNode. But before we can install the software, we first need to make sure we can communicate with the HeadNode using HTTPS. We first need to install the `.cer` file from the HeadNode to make sure the HeadNode and the ComputeNode trust each other. We need to add this certificate to the `Root` of the local machine.

```powershell
$cerFileName = "C:\ProgramData\QRM\HpcHnPublicCert.cer"
Import-Certificate -FilePath $cerFileName -CertStoreLocation Cert:\LocalMachine\Root  
``` 

When we created the certificate, we used a password from the `AWS Parameter Store` to export it from the certificate store on the HeadNode. We now need the same password to install the software on the ComputeNode. We also need to tell the ComputeNode what the HeadNode is its needs to connect to. So we change the command line arguments a bit and then call the `Start-Process` again.

```powershell
$tgtdir = "C:\ProgramData\QRM\HPC\2016\"
$certpath = "C:\ProgramData\QRM\HPCCertificate.pfx"
$certificate_password = aws ssm get-parameter --name "/certificate/hpc/password" --query "Parameter.Value" --with-decryption --output text --region eu-central-1

$dnsname = aws ssm get-parameter --name " /Vpc/Default/PrivateDns/Name" --query "Parameter.Value" --output text --region eu-central-1
$HeadNode = "HeadNode.mydomain.com"

$setupArg = "-unattend -ComputeNode:$HeadNode -SSLPfxFilePath:$certpath -SSLPfxFilePassword:$certificate_password"

$p = Start-Process -FilePath "$tgtdir\setup.exe" -ArgumentList $setupArg -PassThru -Wait
if($p.ExitCode -eq 0)
{
    LogWrite "Succeed to Install HPC Pack HeadNode"
    break
}
if($p.ExitCode -eq 3010)
{
    LogWrite "Succeed to Install HPC Pack HeadNode, a reboot is required."
    break
}
```

### Double-hop

After installing the HPC Pack on the ComputeNode we need to make sure it can connect to the HeadNode. We run the PowerShell from a pipeline; there is a remote connection to the EC2 instance with credentials to connect. But we need different credentials to connect to configure the HeadNode, so we need to provide different credentials. Because we use credentials to configure the ComputeNode which are passed thru to the HeadNode we get a `double-hop` issue. The standard in remote PowerShell is that you can ony use the credentials on the machine you are connected to, you are not allowed to connect to another machine. 

To configure the ComputeNode correctly, we need tell PowerShell it is allowed to pass the credentials to the HeadNode. To make this work, we need to create a `SessionConfiguration`; here we can provide the correct credentials by using the `-RunAsCredential`. Next, we can call the script to configure the ComputeNode using the `SessionConfiguration`, now this script will always use the provided credentials also when the connection to the HeadNode.

```powershell
Register-PSSessionConfiguration -Name InstallHPC -RunAsCredential $credential -Force 
Invoke-Command -ComputerName . -Credential $credential -File C:\\programdata\\QRM\\configure_hpc_ComputeNode.ps1 -ConfigurationName InstallHPC
```

### Configure the ComputeNode

The script for configuring the ComputeNode will use the PowerShell module for HPC. When installing the ComputeNode it will install all the software needed to run a ComputeNode. What we need to do is set the node from `unapproved` to `online`. First, we use the `Get-HpcNode` to get information about the node. Then we assign the template we need for this ComputeNode. After assigning the template the node status will change to `Offline`, now we can set the node state to `Online`. IF you start the HPC software on the HeadNode you will see this ComputeNode as `Online`, and it can be used by the HeadNode to run jobs.

```powershell
$DefaultNodeTemplate = "Default ComputeNode Template"

Add-PSSnapin Microsoft.HPC

$node = Get-HpcNode -HealthState Unapproved -Name $env:COMPUTERNAME
Assign-HpcNodeTemplate -Name $DefaultNodeTemplate -Node $node -PassThru -Confirm:$false -ErrorAction SilentlyContinue -WarningAction SilentlyContinue -ErrorVariable Err
   
$node = Get-HpcNode -HealthState OK -State Offline -Name $env:COMPUTERNAME
Set-HpcNodeState -Name $env:COMPUTERNAME -State Online -ErrorAction Stop
```

### Setting up CloudFormation template yaml

Just like the SQL Server and the HeadNode we will also use an `AutoScalingGroup` from running the ComputeNodes. Because you probably want more then one ComputeNode we will not assign IP addresses or DNS entries, simply because we do not really care about this configuration on the ComputeNodes and because we do not need them.  

We again just just change the `UserData` option in the CloudFormation file. In the `UserData` part, we need to put getting the credentials we need from the `AWS Parameter Store` and make sure we use the copy certificate script before installing the HPC Pack. We separated the installation and the configuration from the ComputeNode into two scripts. Only for the configuration, we need the `PSSessionConfiguration`, for the installation, we can just pass tru the credentials using the `Invoke-Command` method.

```yaml
UserData:
  Fn::Base64: 
    Fn::Sub: |
      <powershell>
        $ServiceusernameAccount = aws ssm get-parameter --name "/qrm/service-account/username" --query "Parameter.Value" --with-decryption --output text --region eu-central-1
        $password = aws ssm get-parameter --name "/qrm/service-account/password" --query "Parameter.Value" --with-decryption --output text --region eu-central-1
        $password = $password | ConvertTo-SecureString -AsPlainText -Force
        $credential = New-Object System.Management.Automation.PSCredential($username,$password)

        New-PSDrive -Name "InstallDir" -PSProvider "FileSystem" -Root "\\HeadNode.${PrivateHostedZoneName}\REMINST" -Credential $credential
        Copy-Item InstallDir:\certificates\*.* -Destination "C:\ProgramData\QRM"

        Invoke-Command -ComputerName . -Credential $credential -File C:\\programdata\\QRM\\install_hpc_ComputeNode.ps1 

        Register-PSSessionConfiguration -Name InstallHPC -RunAsCredential $credential -Force 
        Invoke-Command -ComputerName . -Credential $credential -File C:\\programdata\\QRM\\configure_hpc_ComputeNode.ps1 -ConfigurationName InstallHPC

        Remove-PSDrive -Name InstallDir
      </powershell>
      <persist>true</persist>
```

# Dependencies

The tricky part of installing this HPC Pack is the dependency between all the different components. You can indicate dependencies between the different resources, but there is a small problem here because when the EC2 instances is started the CloudFormation resource is done, but we are not done with the software installation yet. To resolve this problem, you can use the `cfn-signal.exe` functionality to indicate the installation is completed. You can use this to send a signal when the software installation is done. What you need to do is change the CloudFormation template and put all the scripts in the `Metadata` option

Within the `MetaData` you need to configure `configsets` which need to contain the script you want to run. You can use multiple commands that need to run after each other. The last step is sending the signal when done. In the resource that depends on this resource, add the `WaitOnResourceSignals: true` in the `UpdatePolicy` section in the CloudFormation file.

```yaml
Metadata:
  AWS::CloudFormation::Init:
    configSets:
      ascending:
        - setup
        - install_hpc
    setup:
      files:
        c:\ProgramData\installdata\install_hpc.ps1:
          content: !Sub |
            New-PSDrive -Name "InstallDir" -PSProvider "FileSystem" 
                -Root "\\HeadNode.${PrivateHostedZoneName}\REMINST" -Credential $credential
            Copy-Item InstallDir:\certificates\*.* -Destination "C:\ProgramData\QRM"
    install_hpc:
      commands:
        00-add-adminuser:
          command: powershell.exe -ExecutionPolicy Unrestricted Add-LocalGroupMember 
            -Group "Administrators" -Member "awsad\$env:computername$"
        01-install-hpc:
          command: powershell.exe -ExecutionPolicy Unrestricted C:\programdata\qrm\install_hpc.ps1
        02-signal-completion:
          command: !Sub >
            cfn-signal.exe -e %ERRORLEVEL% --resource ComputeNodeWindowsAutoScalingGroup
            --stack ${AWS::StackName} --region ${AWS::Region}
```

# Done!!

So that's it, we create a High Performance Cluster running on Amazon Web Services without using the AWS Console and without manual actions. We can use these scripts and templates every time we need a new configuration and we can deploy it to every environment. Because we use an `AutoScalingGroup` for every instance we created, AWS will make sure we have the minimum number of instances running.
