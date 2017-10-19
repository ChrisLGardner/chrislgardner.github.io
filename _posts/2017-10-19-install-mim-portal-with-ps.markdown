---
layout: post
title:  "Install MIM Portal with PowerShell"
categories: powershell
date:   2017-10-18 12:00:00 +0100
excerpt_separator: <!--more-->
---

Following on from my [earlier post]({{ site.url }}/powershell/2017/09/21/install-mim-sync-with-ps.html) about installing MIM Sync I've moved on to installing MIM Portal and Service via PowerShell DSC.
<!--more-->
MIM Portal and Service have some pretty big dependencies, the major one being SharePoint 2013 or 2016. Luckily for me the Azure marketplace has an image with this already installed but not configured, this saved me a good amount of time downloading the ISO, unpacking it and installing it, which proved to be very important when Azure has a 90 minute timeout on the DSC extension.

## Prerequisites ##

MIM also depends on SQL, not a problem as we've already got a dedicated SQL server for Sync and SharePoint to use, but it does the dependency in a dumb way by requiring it's "installed" on the machine running the installer. You can get around this by installing SQL Server Management Studio, I dropped the ISO into Azure Files to speed up downloading it so I can use Copy-Item rather than Invoke-WebRequest (which is very slow for large files due to caching them in memory while downloading).

You can access Azure Files pretty easily as a PSDrive using a simple Script block to connect:

{% highlight PowerShell linenos %}
Script DownloadSSMS {
    GetScript = { @{} }
    TestScript = {Test-Path -Path 'C:\Packages\SSMS-Setup-ENU.exe'}
    SetScript = {
        $FileSystemCredential = New-Object System.Management.Automation.PSCredential ("AZURE\<UserName>", (ConvertTo-SecureString "<AccessKey>" -AsPlainText -Force))
        New-PSDrive -Name Q -PSProvider Filesystem -Root \\<StorageAccount>.file.core.windows.net\<Container> -Credential $FileSystemCredential
        Copy-Item -path "Q:\SSMS-Setup-ENU.exe" -Destination C:\Packages
    }
}

Package InstallSSMS {
    Name = 'SQL Server Management Studio'
    ProductId = 'CD1FA99A-EEF9-44BE-8A89-8FB17F1C5437'
    Path = "C:\Packages\SSMS-Setup-ENU.exe"
    Arguments = "/install /quiet /norestart /log c:\Packages\ssms.log"
    PsDscRunAsCredential = $DomainCredentials
    Ensure = 'Present'
    DependsOn = '[Script]DownloadSSMS'
}
{% endhighlight %}

Ideally you'd store the access key in KeyVault or similar and pull it out at deploy time as a secure string but I'm showing the alternative method here for demo purposes. I chose to copy to C:\Packages as I know that folder will exist at run time as Azure uses it to store all the extensions that it installs and there will always be at least 1 of those, the DSC extension used here.

The documentation for installing MIM Portal and it's prerequisites assume you are using [SharePoint 2013](https://docs.microsoft.com/en-us/microsoft-identity-manager/prepare-server-sharepoint), however a [helpful blogger](http://www.mim.ninja/2017/08/17/installing-mim-on-sharepoint-2016/) linked a post on the changes needed for SharePoint 2016. **One caveat of using SharePoint 2016 is that you will need to use MIM SP1.**

As detailed in that blog you'll need to set the Compatability Level of the site to 15, SharePointDSC can handle this quite easily as part of the creation of the site:

{% highlight PowerShell linenos %}
SPSite MIMPortalHostSite {
    Url                      = "https://mim.$DomainName"
    OwnerAlias               = $SharePointAdminCredential.UserName
    Name                     = 'MIM'
    Template                 = "STS#1"
    CompatibilityLevel       = 15
    PsDscRunAsCredential     = $SharePointAdminCredential
    DependsOn                = "[SPWebApplication]MIMPortalWebApp"
}
{% endhighlight %}

The other change to SharePoint that needs to be made is disabling Server Side View, this is usually a simple change using the code examples in either the documentation of the blog post however with DSC this is a little less easy. There is no SharePointDSC options for it, a normal Script block won't do it either as it needs to run as the SharePoint admin, the Credential property on Script just launches an Invoke-Command session as that user but you run into double hop issues that way and due to some issue I never quite diagnosed fully this wouldn't work when using PSDscRunAsCredential. So my solution was to use the Script resource but create my own Invoke-Command session within it and make use of CredSSP (I'd prefer not to but the SharePoint Snapin talks to SQL), we'd enabled CredSSP earlier in our configuration for our other SharePoint resources.

{% highlight PowerShell linenos %}
Script DisableServerSideView {
    GetScript = { @{} }
    TestScript = {
        Test-Path -Path C:\Users\sp2016_admin\Documents\ViewStateOnServer.txt
    }
    SetScript = {
        Invoke-Command -ComputerName $env:computername -ScriptBlock {
            Add-PSSnapin -Name Microsoft.SharePoint.Powershell
            $contentService = [Microsoft.SharePoint.Administration.SPWebService]::ContentService
            $contentService.ViewStateOnServer = $False
            $contentService.Update()
            Set-Content -Path C:\Users\sp2016_admin\Documents\ViewStateOnServer.txt -Value 'View State Changed to False' -force
        }  -Authentication CredSSP -Credential $Using:SharePointAdminCredential
    }
}
{% endhighlight %}

## Install Portal And Service ##

With all that done we finally get to the point of actually installing MIM Portal and Service itself. The [documentation](https://docs.microsoft.com/en-us/microsoft-identity-manager/install-mim-service-portal) for this of course shows all the lovely screenshots with buttons to click and boxes to fill in, not an option via DSC so out comes Orca and a manual install using /l*v logging to figure out what all the various properties I need to tell it are.

{% highlight PowerShell linenos %}
$InstallArguments = @(
    #Core Properties needed
    "TRANSFORM=C:\Packages\portal.mst"
    "FEATURES_TO_INSTALL=ALL"
    "FEATURES_TO_EXCLUDE=PAMServices"
    "SHAREPOINTVERSION=2013OR2016"
    "ENABLE_REPORTING=0"
    "CERTIFICATE_NAME=ForefrontIdentityManager"
    "ALLUSERS=1"
    "STS_PORT=5726"
    "RMS_PORT=5725"
    "ACCEPT_EULA=1"
    "FIREWALL_CONF=1"
    "SQMOPTINSETTING=1"
    "REBOOT=ReallySuppress"
    "/l*v C:\MIM\$(Get-Date -format 'yyyy-MM-dd-hh-mm').log"

    #SQL Server Properties
    "SQLSERVER_SERVER=$SqlNodeName"
    "SQLSERVER_DATABASE=FIMService"
    "EXISTINGDATABASE=0"

    #MIM Service Properties
    "SERVICEADDRESS=$Env:ComputerName"
    "SERVICE_ACCOUNT_NAME=MIMPortalService"
    "SERVICE_ACCOUNT_PASSWORD=$($Credential.GetNetworkCredential().Password)"
    "SERVICE_ACCOUNT_DOMAIN=$DomainNetBiosName"
    "SERVICE_ACCOUNT_EMAIL=MIMPortalService@$DomainName"

    #MIM Sync Properties
    "RUNNING_USER_EMAIL=Administrator@$DomainName"
    "SYNCHRONIZATION_SERVER_ACCOUNT=$DomainNetBiosName\MIMSyncService"
    "SYNCHRONIZATION_SERVER_ACCOUNT_NAME=MIMSyncService"
    "SYNCHRONIZATION_SERVER_ACCOUNT_DOMAIN=$DomainNetBiosName"
    "SERVICE_ACCOUNT_DOMAIN=$DomainNetBiosName"
    "SYNCHRONIZATION_SERVER=$SyncNodeName"
    "SERVICEADDRESS=$Env:Computername"

    #SharePoint Properties
    "SHAREPOINTUSERS_CONF=1"
    "SHAREPOINTTIMEOUT=1440"
    "SHAREPOINT_URL=https://mim.$DomainName"

    #MIM Self Service Password Reset Registration Properties
    "REGISTRATION_ACCOUNT=$DomainNetBiosName\sspr_registration"
    "REGISTRATION_ACCOUNT_DOMAIN=$DomainNetBiosName"
    "REGISTRATION_ACCOUNT_NAME=sspr_registration"
    "REGISTRATION_ACCOUNT_PASSWORD=$($Credential.GetNetworkCredential().Password)"
    "REGISTRATION_PORT=8080"
    "REGISTRATION_SERVERNAME=$Env:ComputerName"
    "REGISTRATION_PORTAL_URL=https://registartion.$DomainName"
    "REGISTRATION_HOSTNAME=registration.$DomainName"
    "IS_REGISTRATION_EXTRANET=Extranet"
    "REGISTRATION_FIREWALL_CONF=1"
    "REGISTRATION_FIREWALL_CONF=1"
    "REGISTRATION_FIREWALLMODE=INSTALL"

    #MIM Self Service Password Reset Properties
    "RESET_ACCOUNT=$DomainNetBiosName\sspr_reset"
    "RESET_ACCOUNT_DOMAIN=$DomainNetBiosName"
    "RESET_ACCOUNT_NAME=sspr_reset"
    "RESET_ACCOUNT_PASSWORD=$($Credential.GetNetworkCredential().Password)"
    "RESET_PORT=8088"
    "RESET_SERVERNAME=$Env:ComputerName"
    "RESET_HOSTNAME=reset.$DomainName"
    "IS_RESET_EXTRANET=Extranet"
    "RESET_FIREWALL_CONF=1"
    "FIREWALLMODE=INSTALL"
    "RESET_FIREWALLMODE=INSTALL"

    #MIM PAM Properties - Not sure if these are needed if we're not installing PAM
    "MIMPAM_ACCOUNT_DOMAIN=$DomainNetBiosName"

    #Exchange Properties
    "MAIL_SERVER=$ExchangeNodeName"
    "MAIL_SERVER_USE_SSL=1"
    "MAIL_SERVER_IS_EXCHANGE=1"
    "POLL_EXCHANGE_ENABLED=1"
)
{% endhighlight %}

As you can see there are a few properties to set. If you're not using the Self Service Password Reset functionality or it's on a different machine then those properties will not be needed and some others will but unfortunately I didn't get those.

The transform file I'm using adds 2 new Properties:

{% highlight PowerShell linenos %}
IS_SYNC_SERVICE_RUNNING = 0
IS_SYNC_SERVICE_EXISTS = 0
{% endhighlight %}

This was due to the MIM Portal server not detecting the Sync server, almost certainly due to a firewall issue but that's next on my list of things to investigate and ensure the ports are open correctly on the Sync server.

### DSC Install ###

With all those properties set and stored in an array we can then use the Package resource in the same way as in the MIM Sync install:

{% highlight PowerShell linenos %}
Package MimPortalInstall {
    Name = 'Microsoft Identity Manager Service and Portal'
    ProductId = '0782FB14-023A-430F-B0D5-4AE1D1CCFCAA'
    Path = "C:\MIM\Service and Portal\Service and Portal.msi"
    Arguments = " $($InstallArguments -join ' ')"
    PsDscRunAsCredential = $SharePointAdminCredential
    Ensure = 'Present'
    DependsOn = '[Script]ExtractMimIso','[SPSite]MIMPortalHostSite'
}
{% endhighlight %}

This installer can take a little while to complete, on its own it takes around 10 minutes but when added to the SharePoint configuration and other things happening in the DSC configuration for this server then it starts to drift very close to the 90 minute timeout. The configuration will still complete correctly but Azure will report it as a failure, which can cause your deployment pipeline to fail if using something like VSTS.

### MSI Exec install ##

Under the hood the Package resource detects that it's an msi being installed and calls msiexec.exe with `/qn` and `/i` switches. Following a similar approach we can install MIM Portal and Service when logged on as the correct user (or using Invoke-Command and CredSSP) with a command line similar to below:

{% highlight PowerShell linenos %}
$StartProcessParams = @{
    Filepath = 'msiexec.exe'
    Argumentlist = @(
        '/i "C:\MIM\Service and Portal\Service and Portal.msi"'
        '/qn'
        $InstallArguments
    )
    Wait = $True
}

Start-Process @StartProcessParams
{% endhighlight %}

This will perform the same installation as DSC and take the same amount of time but won't add the verification that the Package resource adds to ensure the installation completed successfully, for that you'd need to check the log file yourself. For the log file I've found that it's around 1400kb if it was a successful installation and anything more than that usually suggests it's failed to install, probably in the SharePoint section.

## Installation Problems ##

The main area I've found installation problems are with SharePoint, usually with the WSP failling to install for a variety of reasons. The most common was due to another update happening at the same time and OWSTIMER reporting a conflict, I don't know enough about SharePoint to know if it's possible to deal with this in an easy way but I just left it a few minutes and tried again and it would usually work.

One major issue I found was that there were times the install would fail to install the WSP and the retraction wouldn't complete correctly. This led to future attemtps failling for reasons related to the WSP and any attempts to remove it claimed they were successful but MIM still failed to install because it thought the WSP was installed. The only solution I found was to nuke the SQL and Portal servers and start again. Snapshots before applying the DSC helped speed this up rather than redeploying the full environment.

## Conclusion ##

Hopefully this proves itself useful to somebody, it was certainly an interesting experience figuring out all the properties needed and fighting the installer to correctly install MIM Portal manually.

Next blog post will be on something a bit less installation focused but probably still DSC related, I'll have to see what new challenge I have to deal with on the next project I work on.
