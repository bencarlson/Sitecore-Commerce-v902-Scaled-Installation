
# Deploying a Scaled version of Sitecore Commerce

## Overview

Assumes that the Sitecore platform is up and running (see Ramons scripts)

a) Each Role has a separate ps1 script and a matching SIF config JSON.
b) Each ps1 is identical but points to its relevant config JSON file i.e. the parameters are the same. In other words, we setup all the environment variables in the ps1 script then copied them and changed which config JSON it used.

## High level steps

Download Packages for Azure App Service 2018.07-2.2.126
<https://dev.sitecore.net/~/media/36239F63871841BF822BF18DC5E2D536.ashx>
Download Packages for OnPrem:
<https://dev.sitecore.net/~/media/F374366CA5C649C99B09D35D5EF1BFCE.ashx>

1) On one machine copy all the packages required for the install to c:\deploy_xc or another location
2) Replace all the values in the ps1 files to match your environment (all the locations for all the roles), do this for ALL ps1.
3) Copy the ENTIRE folder with all the scripts\deployment packages etc. to the same location on each machine hosting a role. Therefore the you wont have to change paths in the ps1 files.

Copy the distribution files for Sitecore Commerce to:
C:\Sitecore.Commerce.2018.07-2.2.126

In order to do parameter substitution the Microsoft.Web.XmlTransform.dll must be in the deploy directory.  

To acquire this:
Download MSBuild Microsoft Visual Studio Web targets from nuget
<https://www.nuget.org/packages/MSBuild.Microsoft.VisualStudio.Web.targets/>
Do a Manual download
Change extension of downloaded file from “nupkg” to “zip” and extract file “Microsoft.Web.XmlTransform.dll” from path \tools\VSToolsPath\Web in that zip file.
Copy “Microsoft.Web.XmlTransform.dll” file into “c:\deploy_xc\”.

Don’t forget to unblock this file by right clicking, selecting properties and selecting unblock checkbox

You can now run the ps1 on each relevant machine
Once everything is installed using the standard scripts and with the habitat catalog and a sample sxa site start changing certs and things.

## High level script descriptions (in order of installation)

Do a search and replace in all deploy_xc files for "xp902" and replace it with your site name/prefix, i.e. test-<client> or prod-<client>
      
### SSL certificates

Due to the number of SSL certs that are used, and the number of places the thumbprint is used, the best plan for certificates is to use a wildcard cert (i.e. *.hostname.com) and/or SAN (Subject Alternate Name) cert so you can share the same cert (and thumbprint) on all server instances. 

SAN Cert with Wildcard: 
```
New-SelfSignedCertificate -CertStoreLocation Cert:\LocalMachine\My -DnsName "*.domain.com, domain.com, secure.login.domain.com"
```

#### Error: System.Net.WebException: The request was aborted: Could not create SSL/TLS secure channel.
As long as the server certificate is valid, this message is most likely that the Sitecore XP server’s IIS app pool user account does not have read access to the client certificate’s private key. This access is needed so that the Sitecore XP server can encrypt communications using its client certificate.

To remedy this issue, open the local machine certificates (“Manage computer certificates” in a start menu search) on the Sitecore XP server. Find the client certificate (normally under Personal\Certificates). Right click it, choose All Tasks, then Manage Private Keys.... allow full control to the app pool user. 



### Deploy Sitecore Commerce Identity Server

Copy Base VM and create a new VM named "xp902-id"
Rename the machine xp902-id as well

Copy Deploy folder to c:\Deploy_XC
Copy install packages to C:\Sitecore.Commerce.2018.07-2.2.126

Run createrootcert.ps1 to create the root sitecore cert (if selfssl) - **DO NOT DO THIS IF YOU HAVE ALREADY CREATED A ROOT CERT DURING THE XP1 SCALED INSTALL**
**NOTE**: details on manually generating SSL certs is at the end of this document. 

```powershell
.\create-root-cert
```

Install .NET Core Windows Server Hosting 2.0.8:
<https://aka.ms/dotnetcore-2-windowshosting>

Deploy-Sitecore-Commerce-IdentityServer, Run this script to install identity server

```powershell
.\Deploy-Sitecore-Commerce-IdentityServer
```

To get logs to output in the inetpub\<instance>\logs folder, change the IIS 'site' to run as "NETWORK SERVICE", create a logs directory (giving NETWORK SERVICE permission to modify), and in the web.config file, set stdoutLogEnabled="true"

Ensure that the xp902-id site is running IIS and that port and SSL mappings are correct.
Open up port 5050 in the Firewall

Export Self SSL Cert created during install (xp902-id).
This will be installed into bizfx server to allow it to trust the self ssl (only necessary for self ssl!)


### Check all DB connection settings

Search for "TrustedConnection" in the deploy_xc folder and set it to false (this will disable Windows Authentication to the DB. Make sure that the DB URL and credentials are set properly as well. Search for "User Id" as well, as this is another place where Windows Auth sometimes pops up - change it to use SQL Auth. 

### Deploy Commerce Engine Ops Instance

Copy and deploy base VM template
rename machine to: xp902-ce-ops
(requires restart)

Install .NET Core Windows Server Hosting 2.0.8(or latest):
<https://aka.ms/dotnetcore-2-windowshosting>

Copy Deploy folder to c:\Deploy_XC
Copy install packages to C:\Sitecore.Commerce.2018.07-2.2.126
Unzip Sitecore.Commerce.Engine.SDK.2.2.72.zip into
C:\Sitecore.Commerce.2018.07-2.2.126\Sitecore.Commerce.Engine.SDK.2.2.72

Run createrootcert.ps1 to create the root sitecore cert (if selfssl)

```powershell
.\create-root-cert
```

In order to facilitate communication between Sitecore CM/CD and a Commerce Engine, a Cert needs to be created which is installed in both the Commerce Engine machines and the Sitecore Machines.
If you want to use a self generated script for this, update the script below to get the DNS name to match your proposed DNS name for your Commerce Engine service(s).  If commerce services are being run on separate machines, you will need to generate one for each service that will be accessed from Sitecore CM or CD.  This is usually the Shops and Authoring Service.  Minions and Ops services do not need these certs installed.

```powershell
.\InstallCert\createenginecert.ps1
```

Once the certificate is created, export it and keep it available for later install into CM/CD
Deploy-Sitecore-Commerce-Engine - Run this script on each machine hosting a commerce engine. Delete unwanted sites.

In the Module file: c:\deploy_xc\Modules\DeployCommerceDatabase\DeployCommerceDatabase.psm1 - update the UserAccount, connection strings, usernames, etc. These are hardcoded, and use WindowsAuth, not SQL Auth. 

Also, in script, set: [string]$SiteHostHeaderName = "<prefix>-cm",

```powershell
.\Deploy-Sitecore-Commerce-Engine
```

Note that you will see a warning about it being unable to create a user and assign roles in the database.


**DB ERRORS** 
If you get DB errors around DacPac and users/permissions, look at the Module file: DeployCommerceDatabase.psm1 - review the UserAccount, connection strings, usernames, etc. These are hardcoded, and use WindowsAuth, not SQL Auth. 



In config.json for commerce instances (ops, shops, minions and authoring), set: "AntiForgeryEnabled": false
Also, make sure to update the username/password & servername in Global.config, as well as setting "TrustedConnection": false

Open up port 5015 in the firewall

If you are using Self SSL then you will need to export the created SSL cert and install it into the trusted root of the machine that BizFx

Log into previously created Sitecore CM instance and manually create a new role "Commerce DevOps User" and assign Admin to this role.

The next two items do in all commerce instances (Ops, Shops, Authoring and Minions):
1. Go into wwwroot\bootstrap\Global.json and add "sitecore\\Commerce DevOps User" to "AuthorizedRoles" allowed to call "commerceops" api calls. (restart IIS)
2. In wwwroot/data/environments/Plugin.Content.PolicySet-1.0.0.json, open it up and replace the "Host" from sxa.storefront.com to "xp902-cm" (or name you used for CM Service)

Run Bootstrap process to load configuration files.
If you have customized the configuration files then update them before running this process.

```powershell
.\Bootstrap-Commerce-Engine
```

### Deploy Commerce Engine Authoring

Copy and deploy base VM template
rename machine to: xp902-ce-auth
(requires restart)

Install .NET Core Windows Server Hosting 2.0.8(or latest):
<https://aka.ms/dotnetcore-2-windowshosting>

Copy Deploy folder to c:\Deploy_XC
Copy install packages to C:\Sitecore.Commerce.2018.07-2.2.126
Unzip Sitecore.Commerce.Engine.SDK.2.2.72.zip into
C:\Sitecore.Commerce.2018.07-2.2.126\Sitecore.Commerce.Engine.SDK.2.2.72

Run createrootcert.ps1 to create the root sitecore cert (if selfssl)

```powershell
.\create-root-cert
```

Deploy-Sitecore-Commerce-Engine - Run this script on each machine hosting a commerce engine. Delete unwanted sites.

```powershell
.\Deploy-Sitecore-Commerce-Engine
```

Note that you will see a warning about it being unable to create a user and assign roles in the database.

Open up port 5000 in the firewall

If you are using Self SSL then you will need to export the created SSL cert and install it into the trusted root of the machine that BizFx

Install storefront.engine.cer from deployment directory into localmachine/root (TODO: shouldn't it be done in the script?)

Update wwwroot/config.json with thumbprint from cert installed:
‎77e580a6027ffd27c4445c08b666e46dc5adf0db

Export the Self SSL Cert created and add it to deploy_xc/sslcerts (you will use it later)

### Deploy Commerce Engine Shops

Deploy-Sitecore-Commerce-Engine - Run this script on each machine hosting a commerce engine. Delete unwanted sites.

Be sure to download the Sitecore app files and install in: C:\Sitecore.Commerce.2018.07-2.2.126

Deploy-Sitecore-Commerce-CommerceSolrCores, run this on the SOLR server or from a machine with the correct access to the solr cores folder. <- may require updating paths to resource files. 

### Deploy Commerce BizFx

Copy base VM and rename to xp902-bizfx

Copy Deploy folder to c:\Deploy_XC
Copy install packages to C:\Sitecore.Commerce.2018.07-2.2.126

Unzip Sitecore.BizFx.1.2.19.zip to deploy directory (it will copy the files from there)

Run create-root-cert to add root Sitecore certs

```powershell
.\create-root-cert
```

Deploy-Sitecore-Commerce-BizFx - Run this script to install business tools.

```powershell
.\Deploy-Sitecore-Commerce-BizFx
```

Open up port 4200 in the firewall

Copy the Cert you exported from the Identity Service step and install into localmachine/root in order to allow it to trust the self ssl.

Once this is run you should be able to start testing connectivity and authentication

From BizFx, you should be able to get the welcome page of identity service:
<https://xp902-id:5050/>

### Deploy Commerce CM Packages

Note: Installing the packages is often problematic and does not well support starting over.
It is recommended, if you are using a virtualized environment, to take a snapshot of both the server being installed to AND the server where SQL Server resides, or backup in some other form.  This allows you to start over if issues occur during the package installation.

Copy Deploy folder to c:\Deploy_XC

Review CM instance config files for "test-<client>-xconnect", "$prefix-xconnect" or similar, replace with appropriate xconnect instance name (i.e. collection, collectionsearch, reporting, processing, etc.)

In the Deploy_xc folder, create a new folder called "assets"
download the following and put into the assets folder:
Sitecore Experience Accelerator 1.7.1
Sitecore PowerShell Extensions-4.7.2

Install the commerce engine connect certificate installed in previous step into the CM machine in localmachine/my

These will be used during the CM package install process

Copy install packages to C:\Sitecore.Commerce.2018.07-2.2.126

Deploy-Sitecore-Commerce-SitecorePackagesCM, Run this on the CM to install the sitecore packages etc. This takes a VERY long time (10+ minutes):

```powershell
.\Deploy-Sitecore-Commerce-SitecorePackagesCM
```
If it times out, you can change the timeout in this line: Invoke-RestMethod $urlInstallModules -TimeoutSec 1800 in the module file 'SitecoreUtilityTasks.psm1'

Go into your Sitecore CM folder: App_Config/Include/Y.Commerce.Engine and open Sitecore.Commerce.Engine.Connect.config
Update the following settings:
Update the App_Config/Include/Y.Commerce.Engine/Sitecore.Commerce.Engine.Connect.config to add the thumbprint from the previous installed engine connect cert.

      <shopsServiceUrl>https://localhost:5000/api/</shopsServiceUrl>
      <commerceOpsServiceUrl>https://localhost:5000/commerceops/</commerceOpsServiceUrl>

to
      <shopsServiceUrl>https://xp902-ce-auth:5000/api/</shopsServiceUrl>
      <commerceOpsServiceUrl>https://xp902-ce-auth:5000/commerceops/</commerceOpsServiceUrl>

Also check the Authoring, Ops and Minons for instances of 'localhost' in the Plugins.<site>xxxxxx.json files, and change them to the correct server naming (test-xx-ce-auth for instance). 


Restart IIS and log into the Sitecore administrator.
Once logged in, review log for connectivity errors.
During startup, Sitecore communicates with the Commerce Engine to validate catalogs.  If there are connectivity errors, they should show up in the logs. If you get TLS/SSL errors, check that the thumbprint here: C:\inetpub\wwwroot\test-<client>.collectionsearch\App_Config\AppSettings.config is the <servername>.xconnect_client and that the thumbprints match in: C:\inetpub\wwwroot\test-<client>-cm\App_Config\ConnectionStrings.config

Once CM packages are deployed, you should switch back over to the server running the Commerce Engine Ops service and Initialize the Commerce Engine. Edit the CommerceEngine.initialize.json file, and make sure the proper environments are listed... i.e <Site>Authoring or HabitatAuthoring

```powershell
.\Initialize-Commerce-Engine
```

This initialization connects to the Sitecore service to pull additional data.  If this succeeds then you are able to successfully connect to Sitecore.

If initialization fails, you may need to use Postman to boostrap the environments. There is a Bootstrap Postman script provided with the commerce install files. 

Switch back over to Sitecore CM machine to continue the deployment process

Enable CEConnect Data Provider and Generate Catalog Templates

Import the xp902-ce-auth self ssl cert from the sslcerts that you created when installing xp902-ce-auth into local machine/root.  This allows it to trust the self ssl cert when communicating from CM to the Commerce Engine. The thumbprint in Sitecore.Commerce.Engine.Connect.config in the CM instance, should be for the CommerceOps instance SSL cert. 

```powershell
.\Deploy-Sitecore-Commerce-EnableSitecoreCM
```

### Deploy Commerce CD Packages

Switch to CD machine created in Sitecore Scaled deployment step

Copy commerce deployment scripts to c:\deploy_xc
Copy commerce deployment modules into c:\sitecore.commerce.2018.07-2.2.126

Deploy-Sitecore-Commerce-SitecorePackagesCD, Run this on each CD. You may need to switch the role to CM first, run the script then, turn it back to a CD.

Switch to CM role by changing
<add key="role:define" value="ContentDelivery"/>
To
<add key="role:define" value="ContentManagement"/>

in <servername>-cd server, App_config/Sitecore.config, change role:require="ContentDelivery" to role:require="ContentManagement" temporarily:

```
      <sc.variable name="defaultContentDatabaseName" value="master" />
  <sc.variable name="defaultContentDatabaseName" role:require="ContentManagement">
    <patch:attribute name="value">web</patch:attribute>
  </sc.variable>
```

  
Copy the "Master" connection string from a CM instance into the App_Config/ConnectionStrings.config.  This is only temporary for the package deployment.

Also, in App_Config/Sitecore.config, copy the "master" database section from CM to CD machine. 


In C:\inetpub\wwwroot\<servername>-cd\Web.config change ContentDelivery to ContentManagement:

```
  <!-- SUPPORTED SERVER ROLES     
         Specify the roles that you want this server to perform. A server can perform one or more roles. Enter the roles in a comma separated list. The supported roles are:

         ContentDelivery
         ContentManagement
         Processing
         Reporting
         Standalone
          
    Default value: Standalone
    -->
    <add key="role:define" value="ContentManagement"/>
```




```powershell
.\Deploy-Sitecore-Commerce-SitecorePackagesCD
```

After deployment, remove above added configurations and switch back to "ContentDelivery"

Fix some issues with it not naming the indexes in configuration to match your chosen prefix.
In App_Config\Sitecore\Marketing.Operations.xMgmt\Sitecore.Marketing.Lucene.Index.Web.config:
Update index node id "sitecore_marketingdefinitions_web" to "sc9u2-marketingdefinitions_web"

You should be able to pull up the default Sitecore content page:
<http://xp902-cd/>


### Fix indexes in CM
At this point, the indexes likely will not be displayed in CM. To fix this, you'll need to do the following:

<cm-docroot>\AppConfig\Sitecore\ContentSearch\Sitecore.ContentSearch.config 

```
<setting name="ContentSearch.Enabled" value="false" /> <- SET TO TRUE
```

Change file extension:

<cm-docroot>\AppConfig\Sitecore\Marketing.Operations.xMgmt\Sitecore.Marketing.Search.config.disabled to 
<cm-docroot>\AppConfig\Sitecore\Marketing.Operations.xMgmt\Sitecore.Marketing.Search.config

What's interesting is if you just changed the first one, did an iisreset, and tested, the ui blows up on:
No service for type 'Sitecore.Marketing.Search.TypeConverters.IStepStringConverter' has been registered.

This is a case where both have to be changed in order to work.  





### Copy Commerce XConnect Models

Copy models json file from CM machine: xp902-cm\XConnectModels
To
XConnect installation (multiple locations):
xp902.collection\App_data\Models
xp902.collectionsearch\App_data\jobs\continuous\IndexWorker\App_data\Models

From here you can go through the normal process to either install your custom tenant and site or install the default tenant and site using the existing instructions.




## SSL Cert generation

Run variants of this script to generate self-signed SSL certs: 

PS C:\deploy_xc>  New-SelfSignedCertificate -DnsName "server-awesome" -CertStoreLocation "cert:\LocalMachine\My" -KeyAlgorithm RSA -KeyLength 2048 -NotAfter (Get-Date).AddYears(42) -KeyExportPolicy Exportable -Provider "Microsoft Enhanced RSA and AES Cryptographic Provider"
