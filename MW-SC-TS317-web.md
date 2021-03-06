# Azure Information Protection New and Advanced Features
### Microsoft Ready Technical Lab 
## MW-SC-TS317

### Introduction

Estimated time to complete this lab

60 minutes

### Objectives

After completing this lab, you will be able to:

- Configure the Azure Information Protection scanner to discover sensitive data 
- Classify and Protect sensitive data discovered by the AIP Scanner
- Synchronize Sensitivity Labels between AIP and SCC
- Use Azure Log Analytics to display centralized reporting for AIP Usage and Data Risk

### Prerequisites

Before working on this lab, you must have:

- Familiarity using Windows 10
- Familiarity with PowerShell

### Lab machine technology

This lab was developed for use in a structured VM environment with the following characteristics:

- Windows Server 2016 Domain Controller (ContosoDC) (Deployed with contoso.azure root domain)
	- Provisioned On-Premises domain admin account named **Contoso\LabUser** with the password **Pa$$w0rd**
	- Provisioned On-Premises users per the table below
  
	> |User Name|Name|Password|
	> |-----|-----|-----|
	> |AdamS|Adam Smith|pass@word1|
	> |AIPScanner||Somepass1|
	> |AliceA|Alice Anderson|pass@word1|
	> |EvanG|Evan Green|pass@word1|
    > |NuckC|Nuck Chorris|NinjaCat123|

- Member Server (Scanner01) with the software below pre-installed
	- Azure AD Connect (Installed, not configured. Available at https://aka.ms/AADConnect)
	- Azure Information Protection client (1.41.51.0) (Available at https://aka.ms/AIPClient)
	- SQL Server 2016+ (Any version will work for AIP Scanner, but full SQL Enterprise was used for this lab setup due to coexistence of SharePoint Server.  If using SQL Express, make sure to use Scanner01\SQLExpress as the SQL Server name in the Scanner Install PowerShell Script)
	- SharePoint Server 2016 Single Server install
	- Demo PII content deployed to a document library at http://Scanner01/documents and in a fileshare shared as \\Scanner01\documents
	- Test PII content available at https://github.com/kemckinnmsft/AIPLAB/blob/master/Content/PII.zip
	- A users.csv file located at c:\ containing the text below. Remove any spaces between or after lines.


		username,displayname,password
		AdamS,Adam Smith,pass@word1
		AIPScanner,AIPScanner,Somepass1
		alicea,Alice Anderson,pass@word1
		evang,Evan Green,pass@word1
		nuckc,Nuck Chorris,NinjaCat123


- Windows 10 Enterprise Client (CLIENT01)
	- Office 365 ProPlus
	- Azure Information Protection client (1.41.51.0) (Available at https://aka.ms/AIPClient)

Microsoft 365 E5 Tenant credentials will be necessary to run through all of the exercises included in this lab.  If you have access to https://demos.microsoft.com, you may use a tenant provisioned there, or your own trial/POC Microsoft 365 Tenant. Global Admin credentials are required to complete most of these exercises.

In the Demos.Microsoft.com Tenants, AIP is pre-populated with labels matching the structure below.  If you are using your own tenant, you may need to create some of these labels to follow along with all of the steps.

> |Label Name|Description|Sub-Label|Protected|User Rights|
> |-----|-----|-----|-----|-----|
> |Personal|Non-business data, for personal use only.|No|No|N/A|
> |Public|Business data that is specifically prepared and approved for public consumption.|No|No|N/A|
> |General|Business data that is not intended for public consumption. However, this can be shared with external partners, as required. Examples include a company internal telephone directory, organizational charts, internal standards, and most internal communication.|No|No|N/A|
> |Confidential|Sensitive business data that could cause damage to the business if shared with unauthorized people. Examples include contracts, security reports, forecast summaries, and sales account data.|No|No|N/A|
> |Recipients Only|Confidential data that requires protection and that can be viewed by the recipients only.|Yes, of Confidential|Yes|User defined, In Outlook apply Do Not Forward|
> |All Employees|Confidential data that requires protection, which allows all employees full permissions. Data owners can track and revoke content.|Yes, of Confidential|Yes|All members, Co-Owner|
> |Anyone (not protected)|Data that does not require protection. Use this option with care and with appropriate business justification.|Yes, of Confidential|No|N/A|
> |Highly Confidential|Very sensitive business data that would cause damage to the business if it was shared with unauthorized people. Examples include employee and customer information, passwords, source code, and pre-announced financial reports.|No|No|N/A|
> |Recipients Only|Highly Confidential data that requires protection and that can be viewed by the recipients only.|Yes, of Highly Confidential|Yes|User defined, In Outlook apply Do Not Forward|
> |All Employees|Highly Confidential data that requires protection, which allows all employees full permissions. Data owners can track and revoke content.|Yes, of Highly Confidential|Yes|All members, Co-Author|
> |Anyone (not protected)|Data that does not require protection. Use this option with care and with appropriate business justification.|Yes, of Highly Confidential|No|N/A|

### Reporting Errors

This is a living document and was developed from lab content so it is possible that some of the screenshots or references are not perfectly translated.  **Additionally, the steps in this lab are designed around specific lab tenants so may not be applicable or may need to be altered in other test environments.  Please modify as necessary for your environment**.  

If you run into any issues in this document or the instructions do not work because of changes in the Azure interface or client software, please reach out to me by clicking on the link below. I will make every effort to ensure that the instructions contained in this document are up-to-date and relevant but constructive criticism is always appreciated.

https://aka.ms/AIPLabFeedback

---


# Azure Information Protection
[:arrow_left: Home](#introduction)

## Overview

Azure Information Protection (AIP) is a cloud-based solution that can help organizations to protect sensitive information by classifying and (optionally) encrypting documents and emails on Windows, Mac, and Mobile devices. This is done using an organization defined classification taxonomy made up of labels and sub-labels. These labels may be applied manually by users, or automatically by administrators via defined rules and conditions.

The phases of AIP are shown in the graphic below.  

![Phases.png](/Media/Phases.png)

In this lab, we will guide you through addressing all of these phases using some of the newest features of AIP.  We first will perform Discovery using the AIP scanner. We recommend that all customers do this step as it only requires AIP P1 licensing and can help to show customers the risk they are currently facing so they can properly prioritize their security investments. 

We will also show how to use the AIP Scanner in Enforce mode to take advantage of AIP P2 features like Automatic Conditions to help them Classify, Label, and Protect the discovered information easily.

We will help you understand how to Enable and Publish labels in the Security and Compliance Center so they can be used with Mac, Mobile, ISVs (like Adobe PDF), and other unified clients.

Finally, we will demonstrate how to use the new AIP Dashboards to leverage Azure Log Analytics to display actionable information on Usage, Activity, and Data Risk.

![Two overlaying screenshots of the Azure Information Protection scanner's blade in the Azure portal. This blade provides dashboards that consolidate information for all deployed Azure Information Protection scanners, including health status, scan results, classification and policy settings, and more.](/Media/8324-image001.png)

## Objectives

This lab assumes that you are familiar with label and policy creation and that you have seen the operation of conditions in Office applications as these will not be demonstrated.  This lab will use the predefined labels and global policy populated in the demo tenants.

---

# Lab Environment Configuration
[:arrow_left: Home](#introduction)

There are a few prerequisites that need to be set up to complete all the sections in this lab.  This Exercise will walk you through the items below.

- [Azure AD User Configuration](#azure-ad-user-configuration)

- [Redeem Azure Pass](#redeem-azure-pass)

- [Configuring Azure Log Analytics](#configuring-azure-log-analytics)

---
# Azure AD User Configuration
[:arrow_up: Top](#lab-environment-configuration)

In this task, we will create new Azure AD users and assign licenses via PowerShell.  In a procduction evironment this would be done using Azure AD Connect or a similar tool to maintain a single source of authority, but for lab purposes we are doing it via script to reduce setup time.

1. Log into Scanner01 using the password ```Somepass1```
2. Open an **Administrative PowerShell Prompt** and run ```C:\Scripts\AADConfig.ps1```.

1. When prompted for the **Tenant name**, **click in the text box** and enter ```Your Tenant Name```.
1. When prompted, provide the credentials below:

	```Global Admin Username```

	```Global Admin Password``` 
   
	> :memo: We are running the PowerShell code below to create the accounts and groups in AAD and assign licenses for EMS E5 and Office E5. This script is also available at [https://aka.ms/labscripts](https://aka.ms/labscripts) as AADConfig.ps1.
    > 
    > #### Azure AD User and Group Configuration
    > $tenantfqdn = "Your Tenant Name"
    > $tenant = $tenantfqdn.Split('.')[0]
	> 
    > #### Build Licensing SKUs
    > $office = $tenant+":ENTERPRISEPREMIUM"
    > $ems = $tenant+":EMSPREMIUM"
	> 
    > #### Connect to MSOLService for licensing Operations
    > Connect-MSOLService -Credential $cred
	> 
    > #### Remove existing licenses to ensure enough licenses exist for our users
    > $LicensedUsers = Get-MsolUser -All  | where {$_.isLicensed -eq $true}
    > $LicensedUsers | foreach {Set-MsolUserLicense -UserPrincipalName $_.UserPrincipalName -RemoveLicenses $office, $ems}
	> 
    > #### Connect to Azure AD using stored credentials to create users
    > Connect-AzureAD -Credential $cred
	> 
    > #### Import Users from local csv file
    > $users = Import-csv C:\users.csv
	> 
    > foreach ($user in $users){
    > 	
    > #### Store UPN created from csv and tenant
    > $upn = $user.username+"@"+$tenantfqdn
	> 
    > #### Create password profile preventing automatic password change and storing password from csv
    > $PasswordProfile = New-Object -TypeName Microsoft.Open.AzureAD.Model.PasswordProfile 
    > $PasswordProfile.ForceChangePasswordNextLogin = $false 
    > $PasswordProfile.Password = $user.password
	> 
    > #### Create new Azure AD user
    > New-AzureADUser -AccountEnabled $True -DisplayName $user.displayname -PasswordProfile $PasswordProfile -MailNickName $user.username -UserPrincipalName $upn
    > }
    > 
    > Start-Sleep -s 10
	> foreach ($user in $users){
	> 
    > #### Store UPN created from csv and tenant
    > $upn = $user.username+"@"+$tenantfqdn
	> 
    > #### Assign Office and EMS licenses to users
    > Set-MsolUser -UserPrincipalName $upn -UsageLocation US
    > Set-MsolUserLicense -UserPrincipalName $upn -AddLicenses $office, $ems
    > }
	> 
    > #### Assign Office and EMS licenses to Admin user
    > $upn = "admin@"+$tenantfqdn
    > Set-MsolUser -UserPrincipalName $upn -UsageLocation US
    > Set-MsolUserLicense -UserPrincipalName $upn -AddLicenses $office, $ems

---

# Sign up for Azure Subscription

For several of the exercises in this lab series, you will require an active subscription.  You can sign up for a free subscription by following the instructions below.  If you have already used your free trial, you can sign up for a pay-as-you-go subscription as nothing in this lab will incur any charges.  Be aware that if you use the scanner to discover a large amount of data, you may incur Azure charges.

### Creating an Azure free subscription

1. Browse to **https://azure.microsoft.com/en-us/free**
2. Click the **Start Free** button in the middle of the screen.

	>![Start Free](/Media/free.png)
1. Sign in using your **Global Admin credentials** for your test tenant.
1. Fill out the information to sign up for a free Azure subscription.

	>![Sign UP](/Media/signup.png)

---
## Configuring Azure Log Analytics 
[:arrow_up: Top](#lab-environment-configuration)

In order to collect log data from Azure Information Protection clients and services, you must first configure the log analytics workspace.

1. In the Azure portal, type the word ```info``` into the **search bar** and press **Enter**, then click on **Azure Information Protection**. 

	> ![2598c48n.jpg](/Media/2598c48n.jpg)
	
	> :memo: If you do not see the search bar at the top of the portal, click on the **Magnifying Glass** icon to expand it.
	>
	> ![ny3fd3da.jpg](/Media/ny3fd3da.jpg)

	> :memo: You should automatically be logged into the azure portal.  If not, navigate to ```https://portal.azure.com/``` and log in with the credentials below.
	>
	>```Global Admin Username``` 
	>
	>```Global Admin Password```

1. In the Azure Information Protection blade, under **Manage**, click **Configure analytics (preview)**.

1. Next, click on **+ Create new workspace**.

	> ![qu68gqfd.jpg](/Media/qu68gqfd.jpg)
1. In the Log analytics workspace using the values in the table below and click **OK**.

	|||
	|-----|-----|
	|OMS Workspace|**Type a unique Workspace Name**|
	|Resource Group|```AIP-RG```|
	|Location|**East US** (Or a location near the event)|

	> :memo: The OMS **Workspace name** must be **unique across all of Azure**. The name is not relevant for this lab, so feel free to use random characters.

	![Open Screenshot](/Media/5butui15.jpg)
1. Next, back in the Configure analytics (preview) blade, **check the boxes** next to the **workspace** and next to **Enable Content Matches** and click **OK**.

	> ![1547437013585](/Media/1547437013585.png)

	> :memo: Checking the box next to **Enable Content Matches** allows the **actual matched content** to be stored in the Azure Log Analytics workspace.  This could include many types of sensitive information such as SSN, Credit Card Numbers, and Banking Information.  This is important to understand and is why access to this ALA workspace should be locked down appropriately.

1. Click **Yes**, in the confirmation dialog.

	> ![zgvmm4el.jpg](/Media/zgvmm4el.jpg)

---

# Configuring AIP Scanner for Discovery
[:arrow_left: Home](#azure-information-protection)

Even before configuring an AIP classification taxonomy, customers can scan and identify files containing sensitive information based on the built-in sensitive information types included in the Microsoft Classification Engine.  

> ![ahwj80dw.jpg](/Media/ahwj80dw.jpg)

Often, this can help drive an appropriate level of urgency and attention to the risk customers face if they delay rolling out AIP classification and protection.  

In this exercise, we will configure an AIP scanner profile in the Azure portal and install the AIP scanner. Initially, we will run the scanner against repositories in discovery mode.  Later in this lab (after configuring labels and conditions), we will revisit the scanner to perform automated classification, labeling, and protection of sensitive documents. This Exercise will walk you through the items below.

- [AIP Scanner Profile Configuration](#aip-scanner-profile-configuration)
- [AIP Scanner Setup](#aip-scanner-setup)

---
## AIP Scanner Profile Configuration
[:arrow_up: Top](#configuring-aip-scanner-for-discovery)

The new AIP scanner preview client (1.45.32.0) and future GA releases will use the Azure portal central management user interface.  You are now able to manage multiple scanners without the need to sign in to the Windows computers running the scanner, set whether the scanner runs in Discovery or Enforcement mode, configure which sensitive information types are discovered and set repository related settings, like file types scanner, default label etc. Configuration from the Azure portal helps your deployments be more centralized, manageable, and scalable.

> ![ScannerUI](/Media/ScannerUI.png)

To make the admin’s life easier we created a repository default that can be set one time on the profile level and can be reused for all added repositories. You can still adjust settings for each repository in case you have a repository that requires some special treatment. 

The AIP scanner operational UI helps you run your operations remotely using a few simple clicks.  Now you can:

- Monitor the status of all scanner nodes in the organization in a single place
- Get scanner version and scanning statistics
- Initiate on-demand incremental scans or run full rescans without having to sign in to the computers running the scanners

> ![ScannerUI2](/Media/ScannerUI2.png)

In this task, we will configure the repository default and add a new profile with the repositories we want to scan.

1. On Client01, in the Azure Information Protection blade, under **Scanner**, click **Profiles**.

	> ![ScannerProfiles](/Media/ScannerProfiles.png)

	> :memo: If the Azure portal is not already open, navigate to ```https://aka.ms/ScannerProfiles``` and log in with the credentials below.
	>
	> ```Global Admin Username```
	>
	> ```Global Admin Password```

1. In the Scanner Profiles blade, click the **+ Add** button.

1. In the Add a new profile blade, enter ```East US``` for the **Proflie name**.

	> :memo: The default **Schedule** is set to **Manual**, and **Info types to be discovered** is set to **All**.

1. Under **Policy Enforcement**, set the **Enforce** switch to **Off**.

1. Note the various additional settings, but **do not modify them**. Click **Save** to complete initial configuration.

	> :memo: For additional information on the options available for the AIP scanner profile, 

1. Once the save is complete, click on **Configure repositories**.

	> ![Configure Repository](/Media/ConfigRepo.png)

1. In the Repositories blade, click the **+ Add** button.

1. In the Repository blade, under **Path**, type ```\\Scanner01\documents```.

1. Under Policy enforcement, make the modifications shown in the table below.

	|Policy|Value|
	|-----|-----|
	|**Default label**|**Custom**|
	||**Confidential \ All Employees**|
	|**Default owner**|**Custom**|
	||```Global Admin UserName```|

	> ![Repo](/Media/Repo.png)

1. Click **Save**.

1. In the Repositories blade, click the **+ Add** button.

1. In the Repository blade, under **Path**, type ```http://Scanner01/documents```.

1. Leave all policies at Profile default, and click **Save**.

---
## AIP Scanner Setup
[:arrow_up: Top](#configuring-aip-scanner-for-discovery)

In this task we will use a script to install the AIP scanner service and create the Azure AD Authentication Token necessary for authentication.

### Installing the AIP Scanner Service

The first step in configuring the AIP Scanner is to install the service and connect the database.  This is done with the Install-AIPScanner cmdlet that is provided by the AIP Client software.  The AIPScanner service account has been pre-staged in Active Directory for convenience.

1. Switch to Scanner01 and log in using the Credentials below.

	> ```AIP Scanner```
	>
	> ```Somepass1```

1. Open an **Administrative PowerShell Window** and type ```C:\Scripts\Install-ScannerPreview.ps1``` and press **Enter**. 

	> :memo: This script installs the AIP scanner Service using the **local domain user** account (Contoso\\AIPScanner) provisioned for the AIP Scanner. SQL Server is installed locally and the default instance will be used. The script will prompt for **Tenant Global Admin** credentials, the **AIP scanner Profile name**, and finally the AIP Scanner cloud account.  In a production environment, this will likely be the synced on-prem account, but for this demo we created a cloud only account during AAD Configuration earlier in the lab.
	>
	> This script only works if logged on locally to the server as the AIP scanner Service Account, and the service account is a local administrator.  Please see the scripts at https://aka.ms/ScannerBlog for aadditional instructions.

	> :memo:  This script will run the code below. This script is available online as Install-ScannerPreview.ps1 at https://aka.ms/labscripts
	> 
	> Add-Type -AssemblyName Microsoft.VisualBasic
	> 
	> $daU = "contoso\AIPScanner"
	> $daP = "Somepass1" | ConvertTo-SecureString -AsPlainText -Force
	> $dacred = New-Object System.Management.Automation.PSCredential -ArgumentList $daU, $daP
	> 	
	> $gacred = get-credential -Message "Enter Global Admin Credentials"
	> 	
	> Connect-AzureAD -Credential $gacred
	> 	
	> $SQL = "Scanner01"
	> 	
	> $ScProfile = [Microsoft.VisualBasic.Interaction]::InputBox('Enter the name of your configured AIP Scanner Profile', 'AIP Scanner Profile', "East US")
	> 	
	> Install-AIPScanner -ServiceUserCredentials $dacred -SqlServerInstance $SQL -Profile $ScProfile
	> 	
	> $Date = Get-Date -UFormat %m%d%H%M
	> $DisplayName = "AIPOBO" + $Date
	> $CKI = "AIPClient" + $Date
	> 	
	> New-AzureADApplication -DisplayName $DisplayName -ReplyUrls http://localhost
	> $WebApp = Get-AzureADApplication -Filter "DisplayName eq $DisplayName"
	> New-AzureADServicePrincipal -AppId $WebApp.AppId
	> $WebAppKey = New-Guid
	> $Date = Get-Date
	> New-AzureADApplicationPasswordCredential -ObjectId $WebApp.ObjectID -startDate $Date -endDate $Date.AddYears(1) -Value $WebAppKey.Guid -CustomKeyIdentifier $CKI
	> 	
	> $AIPServicePrincipal = Get-AzureADServicePrincipal -All $true | Where-Object { $_.DisplayName -eq $DisplayName }
	> $AIPPermissions = $AIPServicePrincipal | Select-Object -expand Oauth2Permissions
	> $Scope = New-Object -TypeName "Microsoft.Open.AzureAD.Model.ResourceAccess" -ArgumentList $AIPPermissions.Id, "Scope"
	> $Access = New-Object -TypeName "Microsoft.Open.AzureAD.Model.RequiredResourceAccess"
	> $Access.ResourceAppId = $WebApp.AppId
	> $Access.ResourceAccess = $Scope
	> 	
	> New-AzureADApplication -DisplayName $CKI -ReplyURLs http://localhost -RequiredResourceAccess $Access -PublicClient $true
	> $NativeApp = Get-AzureADApplication -Filter "DisplayName eq $CKI"
	> New-AzureADServicePrincipal -AppId $NativeApp.AppId
	> 	
	> Set-AIPAuthentication -WebAppID $WebApp.AppId + -WebAppKey $WebAppKey.Guid -NativeAppID $NativeApp.AppId
	>
	> Restart-Service AIPScanner
	> Start-AIPScan

1. When prompted, enter the Global Admin credentials below:

	> ```Global Admin Username```
	>
	> ```Global Admin Password```

1. In the popup box, click **OK** to accept the default Profile value **East US**.

1. When prompted, enter the AIP Scanner cloud credentials below:

	> ```AIPScanner@Your Tenant Name```
	>
	> ```Somepass1```

1. In the Permissions requested window, click **Accept**.

    > ![nucv27wb.jpg](/Media/nucv27wb.jpg)

	> :memo: An AIP scanner Discovery scan will start directly after aquiring the application access token.

	> :warning: If you see a **.NET exception**, press **OK**. This is due to SharePoint startup in the VM environment. This event **must be acknowledged** to complete the discovery scan.

---

# Defining Automatic Conditions 
[:arrow_left: Home](#azure-information-protection)

One of the most powerful features of Azure Information Protection is the ability to guide your users in making sound decisions around safeguarding sensitive data.  This can be achieved in many ways through user education or reactive events such as blocking emails containing sensitive data. 

However, helping your users to properly classify and protect sensitive data at the time of creation is a more organic user experience that will achieve better results long term.  In this task, we will define some basic recommended and automatic conditions that will trigger based on certain types of sensitive data.

1. Switch to Client01 and log in with the password Pa$$w0rd.

1. In the **AIP blade**, under **Analytics** on the left, click on **Data discovery (Preview)** to view the results of the discovery scan we performed previously.

	> ![Dashboard.png](/Media/Dashboard.png)

	> :memo: Notice that there are no labeled or protected files shown at this time.  This uses the AIP P1 discovery functionality available with the AIP Scanner. Only the predefined Office 365 Sensitive Information Types are available with AIP P1 as Custom Sensitive Information Types require automatic conditions to be defined, which is an AIP P2 feature.

	> :warning: If no data is shown, it may still be processing. Continue with the lab and come back to see the results later.

1. Under **Classifications** on the left, click **Labels** then expand **Confidential**, and click on **All Employees**.

	![Open Screenshot](/Media/jyw5vrit.jpg)
1. In the Label: All Employees blade, scroll down to the **Configure conditions for automatically applying this label** section, and click on **+ Add a new condition**.

	> ![cws1ptfd.jpg](/Media/cws1ptfd.jpg)
1. In the Condition blade, in the **Select information types** search box, type ```EU``` and check the boxes next to the **items shown below**.

	> ![xaj5hupc.jpg](/Media/xaj5hupc.jpg)

1. Click **Save** in the Condition blade and **OK** to the Save settings prompt.

	![Open Screenshot](/Media/41o5ql2y.jpg)
1. In the Labels: All Employees blade, in the **Configure conditions for automatically applying this label** section, click **Automatic**.

1. Click **Save** in the Label: All Employees blade and **OK** to the Save settings prompt.

	![Open Screenshot](/Media/rimezmh1.jpg)
1. Press the **X** in the upper right-hand corner to close the Label: All Employees blade.

	![Open Screenshot](/Media/em124f66.jpg)
1. Next, expand **Highly Confidential** and click on the **All Employees** sub-label.

	![Open Screenshot](/Media/2eh6ifj5.jpg)
1. In the Label: All Employees blade, scroll down to the **Configure conditions for automatically applying this label** section, and click on **+ Add a new condition**.

	![Open Screenshot](/Media/8cdmltcj.jpg)
1. In the Condition blade, in the search bar type ```credit``` and check the box next to **Credit Card Number**.

	![Open Screenshot](/Media/9rozp61b.jpg)
1. Click **Save** in the Condition blade and **OK** to the Save settings prompt.

	![Open Screenshot](/Media/ie6g5kta.jpg)
1. In the Labels: All Employees blade, in the **Configure conditions for automatically applying this label** section, click **Automatic**.

	> :memo: The policy tip is automatically updated when you switch the condition to Automatic.
	>
	> ![245lpjvk.jpg](/Media/245lpjvk.jpg)

1. Click **Save** in the Label: All Employees blade and **OK** to the Save settings prompt.

	![Open Screenshot](/Media/gek63ks8.jpg)

1. Press the **X** in the upper right-hand corner to close the Label: All Employees blade.

	![Open Screenshot](/Media/wzwfc1l4.jpg)

---

# Security and Compliance Center 
[:arrow_left: Home](#azure-information-protection)

In this exercise, we will migrate your AIP Labels and activate them in the Security and Compliance Center.  This will allow you to see the labels in Microsoft Information Protection based clients such as Office 365 for Mac and Mobile Devices.

Although we will not be demonstrating these capabilities in this lab, you can use the tenant information provided to test on your own devices.

---
## Activating Unified Labeling
[:arrow_up: Top](#security-and-compliance-center)

In this task, we will activate the labels from the Azure Portal for use in the Security and Compliance Center.

1. On Client01, in the AIP blade, click on **Unified labeling (Preview)**.

	> ![Unified Labeling](/Media/Unified.png)

3. Click **Activate** and **Yes**.

	> ![o0ahpimw.jpg](/Media/o0ahpimw.jpg)

	>:memo: You should see a message similar to the one below.
	>
	> ![SCCMigration.png](/Media/SCCMigration.png) 

1. In a new tab, browse to ```https://protection.office.com/``` and click on **Classifications** and **Labels** to review the migrated labels. 

	>:memo: Keep in mind that now the SCC Sensitivity Labels have been activated, so any modifications, additions, or deletions will be syncronised to Azure Information Protection in the Azure Portal. There are some functional differences between the two sections (DLP in SCC, HYOK & Custom Permissions in AIP), so please be aware of this when modifying policies to ensure a consistent experience on clients. 

---
## Deploying Policy in SCC
[:arrow_up: Top](#security-and-compliance-center)

The previous step enabled the AIP labels for use in the Security and Compliance Center.  However, this did not also recreate the policies from the AIP portal. In this step we will publish a Global policy like the one we used in the AIP portal for use with unified clients.

1. In the Security and Compliance Center, under Classifications, click on **Label policies**.

2. In the Label policies pane, click **Publish labels**.

	![Open Screenshot](/Media/SCC01.png)
3. On the Choose labels to publish page, click the **Choose labels to publish** link.

	![Open Screenshot](/Media/SCC02.png)
4. In the Choose labels pane, click the + Add button.

	![Open Screenshot](/Media/SCC03.png)
5. Click the box next to Display name to select all labels, then click the Add button.

	![Open Screenshot](/Media/SCC04.png)
6. Click the Done button.

	![Open Screenshot](/Media/SCC05.png)
7. Back on the Choose labels to publish page, click the Next button.

	![Open Screenshot](/Media/SCC06.png)
8. On the Publish to users and groups page, notice that All users are included by default. If you were creating a scoped policy, you would choose specific users or groups to publish to. Click Next.

	![Open Screenshot](/Media/SCC07.png)
9. On the Policy settings page, select the General label from the drop-down next to Apply this label by default to documents and email.
10. Check the box next to Users must provide justification to remove a label or lower classification label and click the Next button.

	> ![Open Screenshot](/Media/SCC08.png)

11. In the Name textbox, type ```Global Policy``` and for the Description type ```This is the default global policy for all users.``` and click the Next button.

	![Open Screenshot](/Media/SCC09.png)
12. Finally, on the Review your settings page, click the Publish button.

	> ![Open Screenshot](/Media/SCC10.png)

---

# Classification, Labeling, and Protection with the Azure Information Protection Scanner 
[:arrow_left: Home](#azure-information-protection)

The Azure Information Protection scanner allows you to  classify and protect sensitive information stored in on-premises CIFS file shares and SharePoint sites using an automated process.  This is ideal for protecting large datastores with sensitive information that is not centrally located.  Most customers will prefer this option over moving and manually protecting sensitive documents.  

In this exercise, we will run the AIP Scanner in enforce mode to classify and protect the identified sensitive data. This Exercise will walk you through the items below.

- [Enforcing Configured Rules](#enforcing-configured-rules)
- [Reviewing Protected Documents](#reviewing-protected-documents)
- [Reviewing the Dashboards](#reviewing-the-dashboards)

---

## Enforcing Configured Rules 
[:arrow_up: Top](#classification-labeling-and-protection-with-the-azure-information-protection-scanner)

In this task, we will modify the AIP scanner Profile to enforce the conditions we set up and have it run on all files using the Start-AIPScan command.

1. On Client01, return to **Scanner > Profiles** in the Azure Portal.

	> :memo: If needed, navigate to ```https://aka.ms/ScannerProfiles``` and log in with the credentials below:
	>
	> ```Global Admin Username```
	>
	> ```Global Admin Password```

2. Click on the **East US** profile.

1. In the East US profile, under Profile settings, configure the settings in the table below.

	|**Policy**|**Value**|
	|-----|-----|
	|**Schedule**|**Always**|
	|**Info types to be discovered**|**Policy only**|
	|**Enforce**|**On**|
	
	> ![Enforce](/Media/Enforce.png)

	> :memo: These settings will cause the scanner to run continuously on the repositories, make the scanner only look for the sensitive information types we defined in conditions, and Enforce the labeling and protection of files based on those conditions. Leave all other settings in their current state.

1. Click **Save** then click the **X** to close the blade.

1. Next, under Scanner, click on **Nodes**.

	> ![Nodes](/Media/Nodes.png)

1. Highlight the row containing **Scanner01.Contoso.Azure**, and click **Scan now** in the command list above.

	> ![ScanNow](/Media/ScanNow.png)

1. The previous command can take up to 5 minutes to run on the AIP scanner Server. Follow the commands below to accelerate the process.

	1. Switch to Scanner01 and log in with the password Somepass1.

	1. In an Administrative PowerShell window, run the ```Start-AIPScan``` command.

---

## Reviewing Protected Documents 
[:arrow_up: Top](#classification-labeling-and-protection-with-the-azure-information-protection-scanner)

Now that we have Classified and Protected documents using the scanner, we can review the documents to see their change in status.

1. Switch to Client01 and log in with the password Pa$$w0rd.

2. Navigate to ```\\Scanner01.contoso.azure\documents```. 

	> If needed, use the credentials below:
	>
	>```Contoso\LabUser```
	>
	>```Pa$$w0rd```

	![Open Screenshot](/Media/hipavcx6.jpg)
3. Open one of the Contoso Purchasing Permissions documents.
1. When prompted, provide the credentials below:

	> ```EvanG@Your Tenant Name```
	>
	> ```pass@word1```

1. Click **Yes** to allow the organization to manage the device.
	
	> :memo: Observe that the document is classified as Highly Confidential \ All Employees. 
    >
    > ![s1okfpwu.jpg](/Media/HCAE.jpg)

4. Next, in the same documents folder, open one of the pdf files.
5. When prompted by Adobe, enter ```EvanG@Your Tenant Name``` and press **Next**.
6. Check the box to save credentials and press **Yes**.
1. Click **Accept** in the **Permissions requested** dialog.

	> :memo: The PDF will now open and display the sensitivity across the top of the document.
	>
	> ![PDF](/Media/PDF.png)

	> :memo: The latest version of Acrobat Reader DC and the MIP Plugin have been installed on this system prior to the lab. Additionally, the sensitivity does not display by default in Adobe Acrobat Reader DC.  You must make the modifications below to the registry to make this bar display.
	>
	> In **HKEY_CURRENT_USER\Software\Adobe\Acrobat Reader\DC\MicrosoftAIP**, create a new **DWORD** value of **bShowDMB** and set the **Value** to **1**.
	>
	> ![1547416250228](/Media/1547416250228.png)

---
## Reviewing the Dashboards 
[:arrow_up: Top](#classification-labeling-and-protection-with-the-azure-information-protection-scanner)

We can now go back and look at the dashboards and observe how they have changed.

1. On Client01, open the browser that is logged into the Azure Portal.

	> :warning: Some of the content shown in this dashboard will not be present because we did not perform manual labeling.  This content has been left in to show the capabilities of the reports.

1. Under **Analytics**, click on **Usage report (Preview)**.

	> :memo: Observe that there are now entries from the AIP scanner, File Explorer, Microsoft Outlook, and Microsoft Word based on our activities in this lab. 
	>
	> ![Usage.png](/Media/newusage.png)

2. Next, under Analytics, click on **Activity logs (preview)**.
   
    > :memo: We can now see activity from various users and clients including the AIP Scanner and specific users. 
	>
	> ![activity.png](/Media/activity.png)
	
1. Select the drop-down list under **Labels** and check the box next to **Highly Confidential \ All Employees**. 

	> ![activity2.png](/Media/activity2.png)

1. Click on one of the entries to bring up the **Activity Details** panel.

	> :memo: In the Activity Details panel, you can see all of the details related to the classification, labeling, and protection of the file. The level of detail shown below is only available if you checked the box to Enable document content matches under Configure analytics (Preview). 
	>
	> ![activity2.png](/Media/activity3.png)	

3. Finally, click on **Data discovery (Preview)**.

	> :memo: In the Data discovery dashboard, you can see a breakdown of how files are being protected and locations that have sensitive content.
	>
	> ![Discovery.png](/Media/Discovery2.png)
	> 
	> If you click on one of the locations, you can drill down and see the content that has been protected on that specific device or repository.
	>
	> ![discovery2.png](/Media/discovery2b.png)
	

# AIP Lab Complete 
[:arrow_left: Home](#azure-information-protection)

Congratulations! You have completed the Azure Information Protection Hands on Lab. 

In this lab, you have successfully completed the exercises below covering the four phases of the Information Protection Lifecycle.  We first Discovered and Classified information using the AIP scanner. We showed how to enable unified labeling for use with Mac, Mobile, and 3rd party applications like Adobe Acrobat.  We then Labeled and Protected documents with the AIP scanner in Enforce mode. Finally, we reviewed the Monitoring using Azure Log Analytics and the new AIP Dashboards.

- Sensitive data Discovery using the AIP scanner and the new cloud UI
- Enabling Sensitivity Labels in the Microsoft 365 Security and Compliance Center
- Publishing a Label Policy in the Microsoft 365 Security and Compliance Center
- Automatic Classification and Protection of sensitive data using the AIP scanner
- Review of protected Office and Adobe PDF documents
- Review of the new Azure Log Analytics Dashboards

We hope you enjoyed this lab! Please fill out the survey and let us know what was valuable and what was not so that we may improve the experience for future labs. Thanks!

> ![cat](/Media/ninjacat.png)

[https://blogs.msdn.microsoft.com/oldnewthing/20160804-00/?p=94025](https://blogs.msdn.microsoft.com/oldnewthing/20160804-00/?p=94025)

