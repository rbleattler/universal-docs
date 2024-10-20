---
description: Learn how to host PowerShell Universal in Azure.
---

# Azure

PowerShell Universal is an ASP.NET Core web application and can be hosted in Windows Azure Web Apps.

## Container Web App

Container web apps allow for using Docker images as web apps in Azure. Within the Azure portal, you can create a Docker Container web app by using the App Services \ Create Web App wizard.

Once you have selected a resource group, assigned a name and selected a compute plan, you will be able to create the web app.

![Container Web App](<../../.gitbook/assets/image (310) (1) (1) (2) (1) (1) (1).png>)

Next, you'll need to deploy the image to your web app. To do so, select the Deployment Center and configure the image to pull. You can either pull a static tagged version (like 2.7.3) or pull the latest and your web app will automatically stay up to date with new PowerShell Universal releases.

![Deployment Center Settings](<../../.gitbook/assets/image (314) (1) (1).png>)

Next, we'll need to configure an Azure Storage Account for file shares to store the data for this web app. Within your storage account, create a new File Share that is transaction optimized.

![Storage Account File Shares](<../../.gitbook/assets/image (309) (1) (1) (1).png>)

Back in your App Service, you can setup a Path mapping to the file share we just created. In my example, I've set the mount path to `/Data`. This is where the configuration and database will be stored.

![Path Mappings](<../../.gitbook/assets/image (313) (1) (1) (1).png>)

Finally, we need to configure PowerShell Universal to store data on our file share. On the Application settings tab, set the `Data__ConnectionString` and `Data__RepositoryPath` to store data on the file share.

![Application Settings](<../../.gitbook/assets/image (312) (1) (1).png>)

Restart the App Service and login to your new PowerShell Universal instance. You'll see that data generated by PowerShell Universal is being stored on the share.

![File Share Content](<../../.gitbook/assets/image (302).png>)

### Forwarded Headers

Containers are typically hosted behind a reverse proxy and will not be aware of the actual external URL without additional configuration. In order to ensure that systems that require the external URL, like OpenID Connect, receive the proper information, you will need to set the `ASPNETCORE_FORWARDEDHEADERS_ENABLED`  environment variable to `true`.

![](<../../.gitbook/assets/image (318).png>)

For more information, you can read this blog post from the [Azure team](https://devblogs.microsoft.com/dotnet/forwarded-headers-middleware-updates-in-net-core-3-0-preview-6/).

## Standard Web App

### Manually Creating a Web App

Within the the Azure Portal, you will need to create a new Web App resource. PowerShell Universal currently requires the .NET 5 runtime stack. You can use either Linux or Windows.

![](<../../.gitbook/assets/image (304) (1) (1) (1) (1) (1).png>)

In this example, we'll use the Azure PowerShell module to deploy the Web App manually.

You'll first need to install Azure PowerShell.

```powershell
Install-Module Az
```

Once installed, you'll need to connect to your subscription.

```powershell
Connect-AzAccount
```

Now, you can download the latest version of PowerShell Universal. In this example, we'll download the latest Windows version.

```powershell
$LatestVersion = Invoke-RestMethod https://imsreleases.blob.core.windows.net/universal/production/version.txt
Invoke-WebRequest "https://imsreleases.blob.core.windows.net/universal/production/$LatestVersion/Universal.win7-x64.$LatestVersion.zip" -OutFile .\Universal.zip
```

Now that we have the Az module configured and the Universal ZIP downloaded, we can deploy the Web App.

```powershell
$Parameters = @{
    Force = $true
    ResourceGroupName = 'psu-demo'
    Name = 'psudemo'
    ArchivePath = '.\Universal.zip'
}
Publish-AzWebApp @Parameters
```

After publishing the Web App, view your PowerShell Universal instance by navigating to the Web App's URL.

#### Persistent Storage

The default `appsettings.json` file will store the database and configuration files in a non-persistent location. You can add environment variables to move them to persistent storage within your web app.

**Data\_\_ConnectionString**

The `Data__ConnectionString` environment variable sets the location of the database. You will have to ensure that you enable a "shared" connection for LiteDB to function properly in Azure. Set the value to the following.

```
filename=D:\home\Data\PowerShellUniversal\database.db;Connection=shared
```

**Data\_\_RepositoryPath**

The `Data__RepositoryPath` environment variable sets the location of the configuration files for Universal. Set the value to the following.

```
D:\home\Data\PowerShellUniversal\Repository
```

### Updating your Web App

When a new version of PowerShell Universal is released, you will need to update the application files for your Web App. We recommend removing the application directory and redeploying the files. The database and configuration files are not stored in the application directory.

You can delete the files for your Web App by using the Kudu command API. Your Kudu credentials use Basic authentication and are the same as your [deployment credentials](https://github.com/projectkudu/kudu/wiki/Deployment-credentials).

To delete all the files in your Web App, issue the following command.

```powershell
$Parameters = @{
   Uri = "https://psudemo.scm.azurewebsites.net/api/command"
   Credential = (Get-Credential)
   Body = (@{
      command = "rd /s /q D:\home\site\wwwroot"
      dir = "D:\home\site\wwwroot"
   } | ConvertTo-Json)
}

Invoke-RestMethod @Parameters
```

Once you've delete the application files, you can redeploy them by running the manual creation steps again.

### Setting Adjustments Required

#### JWT Signing Key

The default JWT signing key is not of a sufficient length and will need to be updated. This can be done in `appsettings.json` or by using environment variables.

#### appsettings.json

```
  "Jwt": {
    "SigningKey": "xXyt9UpJKB4Pb*4$hprd!JJoyOcK4ZOV**O7Hug9&@gYHc$",
    "Issuer": "IronmanSoftware",
    "Audience": "PowerShellUniversal"
  },
```

#### Environment Variable

```
$Env:Jwt__SigningKey = 'xXyt9UpJKB4Pb*4$hprd!JJoyOcK4ZOV**O7Hug9&@gYHc$'
```

#### API URL

Azure Web Apps use a reverse proxy and PowerShell Universal does not detect the external URL appropriately. When running jobs, Universal uses the Management API to automatically look up job, script and schedule information. This means that this will fail if it cannot correctly address the external API URL.

The API URL should be the external HTTP address of your Web App. You can update this in `appsettings.json` or by using an environment variable.

#### appsettings.json

```
"Api": {
 "Url": "https://psudemo.azurewebsites.net"
 },
```

#### Environment Variable

```
$Env:Api__Url = "https://psudemo.azurewebsites.net"
```
