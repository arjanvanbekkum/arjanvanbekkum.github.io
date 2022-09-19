---
layout: post
title: "Why does the failover database not scale when using ARM templates"
date: 2022-09-19
summary: "When you scale a failover database with a wrong template definition, it does not throw any exceptions but also does not scale."
minute: 5
tags: [azure]
---

For highly available applications, you can use GEO replication in Azure. It is available for `Storage accounts`, `Azure key vaults`, and Azure SQL servers`. To use this for `Azure SQL servers`, you need to use `failover groups`. 

Currently, I am working on a project that uses `failover groups` with multiple databases. This is working fine, but we noticed that one of the databases could use a little more DTU. So we decided to scale this database on both the primary and the secondary location. 

## Azure SQL Server

First, let's take a look at the ARM templates for creating a `failover group`. First, we created two `SQL Servers`, one in West Europe and one in North Europe. The snippet for creating these two SQL Servers looks something like this. 

```json
{
    "apiVersion": "2015-05-01-preview",
    "location": "[resourceGroup().Location]",
    "name": "[variables('sqlServerName')]",
    "type": "Microsoft.Sql/servers",
}

{
    "apiVersion": "2017-10-01-preview",
    "location": "northeurope",
    "name": "[variables('sqlServerFailOverName')]",
    "type": "Microsoft.Sql/servers",
}
```
## Azure SQL Database

We need one or more `SQL databases` inside the SQL Servers. In this case, the template for deploying a database looks like this. This database is a sub-resource inside of the SQL Server. This is a database for the primary location, but there needs to be one in the secondary location.

```json
 {
    "type": "databases",
    "kind": "v12.0,user",
    "name": "MyDatabase",
    "apiVersion": "2014-04-01-preview",
    "location": "[resourceGroup().Location]",
    "properties": {
      "edition": "[parameters('edition')]",
      "status": "Online",
      "defaultSecondaryLocation": "North Europe",
      "requestedServiceObjectiveName": "S0"
     },
    "dependsOn": [
      "[resourceId('Microsoft.Sql/servers', variables('sqlServerName'))]"
  ]
}        
```
## Failover groups

Now that we have the SQL Server and the databases, we can connect them using a `failover group`. The snippet for creating a `failover group` looks like this. You define a primary and secondary SQL Server and indicate what database needs to be used. 

```json
{
      "location": "West Europe",
      "apiVersion": "2015-05-01-preview",
      "name": "[concat(variables('sqlServerName'), '/',  variables('sqlServerDatabaseGroup'))]",
      "type": "Microsoft.Sql/servers/failoverGroups",
      "properties": {
        "replicationRole": "Primary",
        "replicationState": "CATCH_UP",
        "partnerServers": [
          {
            "id": "[resourceId('Microsoft.Sql/servers', variables('sqlServerFailOverName'))]",
            "location": "North Europe",
            "replicationRole": "Secondary"
          }
        ],
        "databases": [
          "[resourceId('Microsoft.Sql/servers/databases', variables('sqlServerName'), 'MyDatabase')]",
        ]
      },
    }
```
## Scaling the database part one

As you can see in the template above, the database is an `S0` database. This size was more than enough when the project started, but today with all kinds of new features and the data growing, we need a little more DTU. 

To scale the database, we changed the template to look like this. We updated the `requestedServiceObjectiveName` property into `S1` for the primary database. 

```json
 {
    "type": "databases",
    "kind": "v12.0,user",
    "name": "MyDatabase",
    "apiVersion": "2014-04-01-preview",
    "location": "[resourceGroup().Location]",
    "properties": {
      "edition": "[parameters('edition')]",
      "status": "Online",
      "defaultSecondaryLocation": "North Europe",
      "requestedServiceObjectiveName": "S1"
     },
    "dependsOn": [
      "[resourceId('Microsoft.Sql/servers', variables('sqlServerName'))]"
  ]
}        
```
To scale the secondary database, we did the same thing; we changed the `requestedServiceObjectiveName` property into `S1`.

```json
{
    "type": "databases",
    "kind": "v12.0,user",
    "name": "MyDatabase",
    "apiVersion": "2017-10-01-preview",
    "location": "northeurope",
    "properties": {
      "createMode": "OnlineSecondary",
      "sourceDatabaseId": "[resourceId('Microsoft.Sql/servers/databases', variables('sqlServerName'), 'MyDatabase')]",
      "edition": "[parameters('edition')]",
      "status": "Online",
      "defaultSecondaryLocation": "West Europe",
      "containmentState": 2,
      "readScale": "Disabled",
      "requestedServiceObjectiveName": "S1"
  }
}
```
After the deployment, the primary database is an `S1`, just like expected. But if we look at the secondary database, it is still an `S0`. 

## Scaling the database part two

Let's take a closer look at the template we just deployed. If you look closely at the template, you will notice a small difference. The secondary database has a different template. It is `2017-10-01-preview`. If we now look at the [template definition](https://learn.microsoft.com/en-us/azure/templates/microsoft.sql/2014-04-01/servers/databases?pivots=deployment-language-arm-template) on the Azure portal, you will see it does not support the `requestedServiceObjectiveName` property. 

So, to keep things a little consistent, let's change the template version to the same as the primary `2014-04-01-preview` and set the `requestedServiceObjectiveName` to `S1` 

```json
{
    "type": "databases",
    "kind": "v12.0,user",
    "name": "MyDatabase",
    "apiVersion": "2014-04-01-preview",
    "location": "northeurope",
    "properties": {
      "createMode": "OnlineSecondary",
      "sourceDatabaseId": "[resourceId('Microsoft.Sql/servers/databases', variables('sqlServerName'), 'MyDatabase')]",
      "edition": "[parameters('edition')]",
      "status": "Online",
      "defaultSecondaryLocation": "West Europe",
      "containmentState": 2,
      "readScale": "Disabled",
      "requestedServiceObjectiveName": "S1"
  }
}
```
After the deployment, the primary database is an `S1`, just like expected. But if we look at the secondary database, it is still an `S0`. And the deployment indicates no errors and no warnings. All lights are green. 

## Scaling the database for real

As it turns out, you cannot scale a secondary database with the `2014-04-01-preview` template version. That would explain why the template was initially on a `2017` version. 

To make the scaling work, change the template back to `2017-10-01-preview`, but we are adding the `sku` property this time. 

```json
{
        "type": "databases",
        "kind": "v12.0,user",
        "name": "MyDatabase",
        "apiVersion": "2017-10-01-preview",
        "location": "northeurope",
        "properties": {
          "createMode": "OnlineSecondary",
          "sourceDatabaseId": "[resourceId('Microsoft.Sql/servers/databases', variables('sqlServerName'), 'MyDatabase')]",
          "edition": "[parameters('edition')]",
          "status": "Online",
          "defaultSecondaryLocation": "West Europe",
          "earliestRestoreDate": "2017-08-25T00:00:00Z",
          "containmentState": 2,
          "readScale": "Disabled"
        },
        "sku": {
          "name": "Standard",
          "tier": "Standard",
          "capacity": 20
        },
      }  
```

After the deployment, the primary database is an `S1`, just like expected. If we take a look at the secondary database, it is also an `S1`. Although it now works, it feels strange that the secondary database does not scale, and the primary does when we use the same template version. The guess is that scaling a secondary failover database is not supported in the `2014-04-01-preview` version.