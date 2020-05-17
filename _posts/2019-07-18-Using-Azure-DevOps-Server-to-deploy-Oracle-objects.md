---
layout: post
title: "Using Azure DevOps Server to deploy Oracle Objects"
date: 2019-07-18
summary: "How to deploy your Oracle objects using just powershell and Azure DevOps. By using the Oracle client you will be able to deploy objects and scripts without having to install an extension from the marketplace"
minute: 6
category: Azure DevOps
tags: [Azure, AzureDevOps, PowerShell]
---
CI/CD is the only way to deploy and release changes to production. But what if you are not using SQL Server but Oracle instead, are you able to use Azure DevOps? Of course, you are!


I will try to show you how you can deploy your Oracle changes and place them in an Azure DevOps CI/CD pipeline without having to install an extension from the Marketplace. In this case, we are going to use the PowerShell on Remote machine task. 


First, we need a build definition to get our Oracle changes into the release pipeline. To make this work we have to use .SQL files. We need to export all the objects we want to use to a SQL-file. If you want to deploy your Oracle objects (and you should!), use your Oracle tooling and export the procedure as a file. 


The file should look something like this

```sql
CREATE OR REPLACE PROCEDURE procedure_name()
IS
BEGIN
   // your code goes here
END
```

Because we want to deploy and redeploy the Oracle objects, we need the `OR REPLACE`. If you would only use the `CREATE` statement, the deployment will fail if the object already exists. This statement also goes for all the objects you wish to deploy. 


After exporting all the objects from the Oracle database to files on your system, you have to add them to your version control system (e.g., GIT of TFVC). After adding them to your version control, you have to modify the build and make sure all the files are added as artifacts to your build definition. If not, you will not be able to deploy them in the release pipeline.

## PowerShell

The next step is to create a PowerShell to deploy all these objects to the database. We are going to use the Oracle client, which is installed on the target machine, most Oracle client installation will also include the SQLPlus command line tool. The client will provide us the opportunity to connect to the database and push our changes. You also need to make sure "Remote Powershell" is enabled on the Target machine.  


The PowerShell will be used on all our environments, so there no hardcoded reference to the database or servers. The login to the database will be an input parameter and thus be different for Test, Acceptance, or Production. 


```powershell
[CmdletBinding()] 
param(
    [string]$logon
) 
```

The login parameter has the format `UserName/Password@Database`. Beware of putting hard coded passwords into your pipeline our in your scripts. We have used an Active Directory Account to connect to the database, so no username or password was provided in either the pipeline or the scripts. Make sure you set the "External Identified" flag on the user in the Oracle database if you use an Active Directory user. After creating the user, make sure it has permissions to create objects in the database, you will probably need the DBA role or create a new one if you please.


You will need to take care of another thing. That is you also have to add this PowerShell script to your build definition and your release pipeline. It is a best practice to put all the code in your build, including deployment scrips. You do not want (trust me) to rely on external resources for your deployment. We have added the PowerShell script to a different artifact folder, so to determine the correct Oracle folder we are adding some code to detect the correct path.

```powershell
$invocation = (Get-Variable MyInvocation).Value
$directorypath = Split-Path $invocation.MyCommand.Path
$releaseStagingFolder = (get-item $directorypath ).parent.fullname 
$oracleStagingFolder="$releaseStagingFolder\Oracle\"
```

Let's create our method that will push all the Oracle scripts to the database. Using SQLPlus, we have to make sure we exit after running the script, so we need to wrap the file and add some extra lines to exit the command prompt when done or receiving an error. But we also want the script to stop if the release fails and not deploy anything that will break our tests. Finally, we execute the statement using SQLPlus; if there is an error it will show in the Azure DevOps pipeline with the write-error method

```powershell
function Invoke-SqlPlus($file, $logon) 
{
    Write-Output "$file";
 
    # Wrap script to make sqlplus stop on errors (because Oracle doesn't do this
    # by default). Also exit the prompt after the script has run.
    $lines = Get-Content $file;
    $lines = ,'whenever sqlerror exit sql.sqlcode;' + $lines;
    $lines += 'exit';
 
    $result = $lines | sqlplus -S $logon
 
    if (!$?)
    { 
        write-error "$file $result" 
        Exit 1
    }
}
```

The last thing we need to do is call the method for all the files we just added to the pipeline. We already found the location of the Oracle scripts, so the only thing we need to do is create a loop and go through the files. Because we are using SQL files we add a method for collecting all the .SQL files in the folder. Afterward, loop the files and call the method for executing the SQLPlus command. 

```powershell
function Execute-Scripts($directory, $logon) 
{
    $files = @(Get-ChildItem $directory -Filter *.sql -ErrorAction SilentlyContinue | Sort-Object)
    if ($files.Count -gt 0)
    {
       foreach ($file in $files) 
       {
          Invoke-SqlPlus $file.FullName $logon 
       }
    }
}
```

That is all nice, but you probably have more than one folder (like Functions, Packages, Procedures) and you do not want to change the PowerShell script every time you add some folder to the Oracle folder (I sure did not). To run all this code, let's create the last piece of PowerShell to get all folders recursively and call the Execute-Script function for each folder. This last piece will run all the scripts within the subfolders first and then run the scripts in the root folder.

```powershell
$items = @(Get-ChildItem $oracleStagingFolder -Recurse | ?{ $_.PSIsContainer } )

foreach ($item in $items)
{
    Execute-Scripts $item.FullName $logon
}
   
Execute-Scripts $oracleStagingFolder $logon
```

Now we have created the PowerShell to deploy the Oracle changes there are two more problems we need to solve. The first thing we need is an extra script which compiles all the invalid objects in the schema. Objects will go invalid if you deploy them in the wrong order. I sure did not want to make a hardcoded list of the correct order of objects. A schema compile does figure out the correct order and make the invalid objects valid again. 

## Oracle scripts

In the root Oracle folder, I have added a SQL file with just one line, to compile all the objects within the schema. Because it is in the root folder, it will be executed after all the objects are deployed. 

```sql
EXECUTE DBMS_UTILITY.compile_schema(schema => 'schema_name');
```

The second thing we need to solve is how are we going to add or drop a column or index or even a procedure. What will happen if we add a column to a table and we want to deploy our script multiple times, we have to think of something to fix this. Because if we don't, the release will fail if the column does not exist and we try to drop it or if we want to add it and it already exists.
Oracle has (equal to SQL Server) a possibility to verify if objects already exist, you can use de `ALL_*` views to make a query and check whether the object already exists. For example, use the `ALL_PROCEDURES` and `ALL_TAB_COLS` to verify Stored Procedures and Columns.


The script you create to add a column to a table will have to look something like below. We create a query to see if the column exists; if not, we add a new column; if it does, we do not add it again. This script can be run several times without breaking the pipeline. Of course, you can use the same methodology for indexes, constraints, functions, and procedures.

```sql
DECLARE
  V_COLUMN_EXISTS NUMBER := 0;  
BEGIN
  SELECT COUNT(*) 
  INTO V_COLUMN_EXISTS
  FROM ALL_TAB_COLS
  WHERE OWNER = <OWNER>
  AND TABLE_NAME = <TABLE_NAME>
  AND COLUMN_NAME = <COL_NAME>
;

  IF (V_COLUMN_EXISTS = 0) THEN
    EXECUTE IMMEDIATE 'ALTER TABLE <OWNER>.<TABLE_NAME> ADD <COL_NAME> VARCHAR2(1)';
  END IF;
END;
/
```

In Azure DevOps add the "Powershell on Remote Machine" task to the release pipeline and make sure you copy all the files from the Azure DevOps server to the server containing the Oracle Client. Set the correct path in the PowerShell tasks and your almost done. The last step is to add the logon as a parameter in the Powershell task and you are all done!  

Have fun!
 
