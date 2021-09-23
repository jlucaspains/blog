---
published: false
layout: post
date: 2021-09-23T00:00:00.000Z
sharing: true
categories:
  - DevOps
description: >-
  TBD
---

The expectations surrounding app releases have changed greatly in the last several years. From months-long release cycles to release many times in a day. In part, this is made possible by smaller applications powered by micro services and some of the typical technologies employed with these applications. In particular, Containers bring a lot of new capabilities including easier deployments via orchestration tools such as Kubernetes.

As a developer and team lead, one of the best added benefits of containers is how easy it is to spin up an environment for each change. What is typically called Environment per PR. Basically, you have a new, short-lived, environment dedicated for your changes.

This is beautiful, but, what if you don't or can't use containers? Can you spin up environments like this with good old VMs and IIS?

Yes. But not easily.

I'm sure that what I will be showing here is not directly applicable to you, but it should be a good start for most .NET apps deployed using AzureDevOps Pipelines to IIS and SQL Server. These are the steps:

* Build stage
  * Build, unit test, and create artifacts ready for deploy
* PR Deploy stage
  * Deploy IIS website(s)
    * Create IIS website and app pool
    * Update appsettings.json
    * Copy app files to the right location
  * Create DB
    * Create/update DB using a good backup
    * Apply DB Migrations
  * Add comment to PR

## The main pipeline

The pipeline yaml will look like below. For organization sake, I broke it down into 3 files: main pipeline, build stage, deploy pr stage.

<script src="https://gist.github.com/jlucaspains/3ea5b64f77f27b29bbd3893f19ad5f4a.js"></script>

## Build stage
In this stage we want to build the app(s) and publish any deployables as build artifacts. I will assume that you have your app built and ready to be deployed so I won't provide details on how to do this. For the purposes of this post, you should have your backend, frontend, and db migrations as published artifacts in your build.

## Deploy PR stage
In this stage, we will do everything needed to spin up a new environment. This includes: creating a SQL Server DB, restoring a backup with good test data, creating or updating IIS App Pools and Web Sites, Deploy build stage files to appropriate folders, and finally drop a comment in the PR with the new environment URL. To make things easier to understand, let's talk about each step first and see the final stage yaml at the end.

### Restore a good backup as your PR DB
We can achieve this somewhat easily with a procedure in master DB that you can call from Powershell. You should install the latest [Sql Server PS Module](https://www.powershellgallery.com/packages/SqlServer). Also, the user running your pipeline needs very high level privileges. I will not go into whether this is a good idea or not from a security standpoint, but do tighten your DB security as much as you can.

```sql
use MASTER;
GO
CREATE PROCEDURE [dbo].[spRestorePRBackup] (
    @TargetName VARCHAR(50),
    @SourceDb VARCHAR(50),
    @Path VARCHAR(500)
)
AS BEGIN
    -- Get the list of files from the backup file
    DECLARE @fileListTable TABLE (
        [LogicalName]           NVARCHAR(128),
        [PhysicalName]          NVARCHAR(260),
        [Type]                  CHAR(1),
        [FileGroupName]         NVARCHAR(128),
        [Size]                  NUMERIC(20,0),
        [MaxSize]               NUMERIC(20,0),
        [FileID]                BIGINT,
        [CreateLSN]             NUMERIC(25,0),
        [DropLSN]               NUMERIC(25,0),
        [UniqueID]              UNIQUEIDENTIFIER,
        [ReadOnlyLSN]           NUMERIC(25,0),
        [ReadWriteLSN]          NUMERIC(25,0),
        [BackupSizeInBytes]     BIGINT,
        [SourceBlockSize]       INT,
        [FileGroupID]           INT,
        [LogGroupGUID]          UNIQUEIDENTIFIER,
        [DifferentialBaseLSN]   NUMERIC(25,0),
        [DifferentialBaseGUID]  UNIQUEIDENTIFIER,
        [IsReadOnly]            BIT,
        [IsPresent]             BIT,
        [TDEThumbprint]         VARBINARY(32),
        [SnapshotURL]           NVARCHAR(360)
    )
    INSERT INTO @fileListTable EXEC('RESTORE FILELISTONLY FROM DISK = N''' + @path + '''')

    DECLARE @logicalDataFileName  NVARCHAR(128)
    DECLARE @logicalLogFileName   NVARCHAR(128)
    DECLARE @physicalDataFileName NVARCHAR(260)
    DECLARE @physicalLogFileName NVARCHAR(260)

    -- Adjust the name of the backup files using source and target db names
    -- This is needed to make the DB file names unique
    SELECT TOP 1 @logicalDataFileName = LogicalName, @physicalDataFileName = replace(PhysicalName, @sourceDb,@targetName)
    FROM @fileListTable
    WHERE Type = 'D'

    SELECT TOP 1 @logicalLogFileName = LogicalName, @physicalLogFileName = replace(PhysicalName, @sourceDb,@targetName)
    FROM @fileListTable
    WHERE Type = 'L'

    -- If the DB exists, set it to single user so the restore won't fail
    IF (db_id(@targetName) IS NOT NULL)
    BEGIN
        EXEC('ALTER DATABASE [' + @targetName + '] SET SINGLE_USER WITH ROLLBACK IMMEDIATE')
    END

    -- trigger the restore
    RESTORE DATABASE @targetName FROM DISK = @path WITH FILE = 1, MOVE @logicalDataFileName TO @physicalDataFileName, MOVE @logicalLogFileName TO @physicalLogFileName
    
    -- put the db back in multi user mode
    EXEC('ALTER DATABASE [' + @targetName + '] SET MULTI_USER')
END
```

Then, you can invoke that procedure from Powershell like so:

```powershell
Import-Module SQLServer

# This will check if the DB already exists, if it does, skip the restore
# $DbName variable will be based on your PR number
# The SourceDb is used to rename the original SQL Server files to something unique using the PR number
# This script is using a server share for .bak files. You may change this and have a backup file ready in your SQL Server instead.
Invoke-Sqlcmd -ServerInstance MySQLServer -database master -query "IF(DB_ID('$(DbName)') IS NULL) exec spRestorePRBackup @TargetName = '$(DbName)', @SourceDb = 'MyAppDB', @Path = '\\MyServerShare\dbbackup\PR.BAK'"
```

### Update DB version with PR migrations
By now, most projects are using Db Migrations one way or another. I typically do them the old fashioned way where the developer is responsible for manually creating versioned scripts. If you have the scripts in a folder like I do, [DbUp](https://dbup.github.io/) is a great tool to update your DB. There is also a [Visual Studio Marketplace extension](https://marketplace.visualstudio.com/items?itemName=johanclasson.UpdateDatabaseWithDbUp) for it. 

You may use EF migrations or anything else that suites you though.

### Deploy IIS website(s) from template xml
You can achieve this using [appcmd](https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-r2-and-2012/jj635852(v=ws.11)) and [netsh](https://docs.microsoft.com/en-us/windows-server/networking/technologies/netsh/netsh-contexts) as they should be installed with any IIS version you may use. Below are some commands that can be executed in powershell to create a website:

```cmd
appcmd add apppool /name:MyAppPool
appcmd add site /name:MySite /bindings:"https/mysite.lpains.com:443:" /physicalPath:"C:\inetpub\wwwroot\MySite" /applicationPool:"MyAppPool"
```

You can either create the website and app pool with specific commands like above, or, you can export and import the site and app pool. First, export existing site and app pool:

```cmd
appcmd list apppool TemplateAppPool  /config /xml > c:\apppools.xml
appcmd list site TemplateSite /config /xml > c:\sites.xml
```

Since we want each PR site to be unique, we need to do some templating in the xml files. Just change the app pool and site names to something you can find and replace later via powershell:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<appcmd>
  <APPPOOL APPPOOL.NAME="[SiteName]" PipelineMode="Integrated" RuntimeVersion="" state="Started">
    <add name="[SiteName]" managedRuntimeVersion="">
        <processModel identityType="SpecificUser" userName="user" password="password" />
    </add>
  </APPPOOL>
</appcmd>
```

You can then use some powershell to replace the placeholder before using appcmd to apply the file:

```powershell
# $(siteName) variable must be unique. I recommend using the PR Number in it 
$parsedSite = (Get-Content "c:\sites.xml" -raw) -replace "\[SiteName\]","$(siteName)"

$parsedSite | Set-Content "c:\sites.xml"

$parsedAppPool = (Get-Content "c:\apppool.xml" -raw) -replace "\[SiteName\]","$(siteName)"

$parsedAppPool | Set-Content "c:\apppool.xml"
```

Finally, use cmd and below commands to import the site and app pool:

```cmd
mkdir c:\inetpub\wwwroot\$siteName
appcmd add apppool /in < c:\apppools.xml
appcmd add site /in < c:\sites.xml
```

Finally, appcmd won't assign certificates to HTTPS bindings. We can use netsh for that though:

```powershell
$guid = [guid]::NewGuid().ToString("B")

netsh http add sslcert hostnameport="$siteName.lpains.com:443" certhash=thumbprint certstorename=MY appid="$guid"
```

## Updating configuration files
There is a good chance you keep some settings in the appsettings.json for .NET apps and perhaps in other files for your frontend. You can easily update these settings using pipeline variables and tokenized files. I like to use [Replace Tokens from Guillaume Rouchon](https://marketplace.visualstudio.com/items?itemName=qetza.replacetokens) marketplace extension for this.

Example appsettings.json:

```json
{
    "Logging": {
        "LogLevel": {
            "Default": "Warning"
        }
    },
    "AllowedHosts": "*",
    "ConnectionStrings": {
        "Default": "Data Source=MyDB;Initial Catalog=DbName;Integrated Security=True;"
    }
}
```

Example appsettings.json after tokenization:

```json
{
    "Logging": {
        "LogLevel": {
            "Default": "Warning"
        }
    },
    "AllowedHosts": "*",
    "ConnectionStrings": {
        "Default": "#{ConnectionString}#"
    }
}
```

Make sure the variable exists in your pipeline stage:

```yaml
  variables:
    prId: "$(System.PullRequest.PullRequestId)"
    prName: "PR$(prId)"
    siteName: '$(prName)'
    dbServer: 'LPainsDB'
    dbName: '$(siteName)'
    ConnectionString: 'server=$(dbServer);database=$(dbName);integrated security=sspi;'
```

### Add comment to Pull Request
After the PR is deployed, you should add some instructions to the PR on how to test it. This usually includes an environment URL and anything else relevant for the test. There are many tasks to add comments to PR, but I've used [Pull Request Utils from Joachim Dalen](https://marketplace.visualstudio.com/items?itemName=joachimdalen.pull-request-utils) successfully before.

In below example, we comment the URL for the environment as well as instructions on how to update the hosts file so that DNS resolution works:

```yaml
- task: joachimdalen.pull-request-utils.5c6ec8a1-d04c-44c0-99b8-42dd865b42e8.PullRequestComments@0
  displayName: 'Pull Request Comments'
  inputs:
    action: createOrUpdate
    content: |
      Environment created at https://$(siteName).lpains.com
      
      To access it, you need to update your hosts file. Using a powershell terminal as admin, run the following:
      
      ```powershell
      Add-Content -Path $env:windir\System32\drivers\etc\hosts -Value "`n10.0.0.1`t$(siteName).lpains.com" -Force
      ```
```

## PR Environment teardown
Because your PR Environment is in the intranet, there is a good chance you can't reach it from Azure DevOps. If this is true, you won't be able to use Webhooks to tear down your environment when the PR is closed.

You can achieve this with a bit more of powershell. Use it to read the active PRs and compare to the active environments. If any environment does not have a corresponding open PR, you can tear it down.

```powershell
## remember to run this script as an Admin
Import-Module SQLServer

# get open PRs from your repository
$B64Pat = [Convert]::ToBase64String([System.Text.Encoding]::UTF8.GetBytes("your pat token goes here"))
$header = @{Authorization = "Basic $B64Pat"}
$url = "https://dev.azure.com/your-org/your-project/_apis/git/pullrequests?repositoryId=d26dc76c-6593-4612-acde-4ac37af9cf2f&includeCommits=false&includeWorkItemRefs=false"
$prArray = (Invoke-RestMethod -Uri $url -Method Get -ContentType "application/json" -Headers $header).value

# find the existing environments in the VM
$envs = Get-ChildItem -filter "lpainsPR*" -Directory | Select-Object Fullname, Name

foreach($env in $envs){
     $prId = $env.Name -replace "lpainsPR", ""
     # find the PR by number
     $pr = $prArray | Where-Object { $_.pullRequestId -eq $prId }
     # if a PR is not found for the environment, tear it down!
     if ($null -eq $pr ) {
         Write-Host "Removing PR$prId"

         # delete web sites
         appcmd.exe delete site "$($env.Name)_UI"
         appcmd.exe delete site "$($env.Name)_API"

         # delete app pools
         appcmd.exe delete apppool "$($env.Name)_UI"
         appcmd.exe delete apppool "$($env.Name)_API"

         # delete files
         Remove-Item -Recurse -Force $env.FullName
		 
         # drop database
		     Invoke-Sqlcmd -ServerInstance LPainsDB -database master -query "DROP DATABASE $($env.Name)"
     }
}
```

## Putting everything together
The final pipeline files may look like this:

<script src="https://gist.github.com/jlucaspains/3ea5b64f77f27b29bbd3893f19ad5f4a.js"></script>

<script src="https://gist.github.com/jlucaspains/010b1239b4af869fd8f51417873f2951.js"></script>

<script src="https://gist.github.com/jlucaspains/75fcc580c2e125798eaf8946e91e8bbe.js"></script>

## More information
Here is some very useful content to read how other people have done this:

* https://medium.com/nntech/level-up-your-ci-cd-pipeline-with-pull-request-deployments-780878e2f15a
* https://devblogs.microsoft.com/devops/review-apps-in-azure-pipelines/
* https://samlearnsazure.blog/2020/02/27/creating-a-dynamic-pull-request-environment-with-azure-pipelines/