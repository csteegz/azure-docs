---
title: Silent install Azure AD App Proxy connector | Microsoft Docs
description: Covers how to perform an unattended installation of Azure AD Application Proxy Connector to provide secure remote access to your on-premises apps.
services: active-directory
documentationcenter: ''
author: barbkess
manager: mtillman

ms.service: active-directory
ms.component: app-mgmt
ms.workload: identity
ms.tgt_pltfrm: na
ms.devlang: na
ms.topic: article
ms.date: 01/31/2018
ms.author: markvi
ms.reviewer: harshja
ms.custom: it-pro

---

# Create an unattended installation script for the Azure AD Application Proxy connector

This topic helps you create a Windows PowerShell script that enables unattended installation and registration for your Azure AD Application Proxy connector.

This capability is useful when you want to:

* Install the connector on Windows servers that don't have user interface enabled, or that you can't access with Remote Desktop.
* Install and register many connectors at once.
* Integrate the connector installation and registration as part of another procedure.
* Create a standard server image that contains the connector bits but is not registered.

For the [Application Proxy connector](application-proxy-connectors.md) to work, it has to be registered with your Azure AD directory using a global administrator and password. Ordinarily this information is entered during Connector installation in a pop-up dialog box, but you can use PowerShell to automate this process instead.

There are two steps for an unattended installation. First, install the connector. Second, register the connector with Azure AD. 

## Install the connector
Use the following steps to install the connector without registering it:

1. Open a command prompt.
2. Run the following command, in which the /q means quiet installation. A quiet installation doesn't prompt you to accept the End-User License Agreement.
   
        AADApplicationProxyConnectorInstaller.exe REGISTERCONNECTOR="false" /q

## Register the connector with Azure AD
There are two methods you can use to register the connector:

* Register the connector using a Windows PowerShell credential object
* Register the connector using a token created offline

### Register the connector using a Windows PowerShell credential object
1. Create a Windows PowerShell Credentials object `$cred` that contains an administrative username and password for your directory. Run the following command, replacing *\<username\>* and *\<password\>*:
   
        $User = "<username>"
        $PlainPassword = '<password>'
        $SecurePassword = $PlainPassword | ConvertTo-SecureString -AsPlainText -Force
        $cred = New-Object –TypeName System.Management.Automation.PSCredential –ArgumentList $User, $SecurePassword
2. Go to **C:\Program Files\Microsoft AAD App Proxy Connector** and run the following script using the `$cred` object that you created:
   
        .\RegisterConnector.ps1 -modulePath "C:\Program Files\Microsoft AAD App Proxy Connector\Modules\" -moduleName "AppProxyPSModule" -Authenticationmode Credentials -Usercredentials $cred -Feature ApplicationProxy

### Register the connector using a token created offline
1. Create an offline token using the AuthenticationContext class using the values in this code snippet:

        using System;
        using System.Diagnostics;
        using Microsoft.IdentityModel.Clients.ActiveDirectory;

        class Program
        {
        #region constants
        /// <summary>
        /// The AAD authentication endpoint uri
        /// </summary>
        static readonly Uri AadAuthenticationEndpoint = new Uri("https://login.microsoftonline.com/common/oauth2/token?api-version=1.0");

        /// <summary>
        /// The application ID of the connector in AAD
        /// </summary>
        static readonly string ConnectorAppId = "55747057-9b5d-4bd4-b387-abf52a8bd489";

        /// <summary>
        /// The reply address of the connector application in AAD
        /// </summary>
        static readonly Uri ConnectorRedirectAddress = new Uri("urn:ietf:wg:oauth:2.0:oob");

        /// <summary>
        /// The AppIdUri of the registration service in AAD
        /// </summary>
        static readonly Uri RegistrationServiceAppIdUri = new Uri("https://proxy.cloudwebappproxy.net/registerapp");

        #endregion

        #region private members
        private string token;
        private string tenantID;
        #endregion

        public void GetAuthenticationToken()
        {
            AuthenticationContext authContext = new AuthenticationContext(AadAuthenticationEndpoint.AbsoluteUri);

            AuthenticationResult authResult = authContext.AcquireToken(RegistrationServiceAppIdUri.AbsoluteUri,
                ConnectorAppId,
                ConnectorRedirectAddress,
                PromptBehavior.Always);

            if (authResult == null || string.IsNullOrEmpty(authResult.AccessToken) || string.IsNullOrEmpty(authResult.TenantId))
            {
                Trace.TraceError("Authentication result, token or tenant id returned are null");
                throw new InvalidOperationException("Authentication result, token or tenant id returned are null");
            }

            token = authResult.AccessToken;
            tenantID = authResult.TenantId;
        }


2. Once you have the token, create a SecureString using the token:

   `$SecureToken = $Token | ConvertTo-SecureString -AsPlainText -Force`

3. Run the following Windows PowerShell command, replacing \<tenant GUID\> with your directory ID:

   `.\RegisterConnector.ps1 -modulePath "C:\Program Files\Microsoft AAD App Proxy Connector\Modules\" -moduleName "AppProxyPSModule" -Authenticationmode Token -Token $SecureToken -TenantId <tenant GUID> -Feature ApplicationProxy`

## Next steps 
* [Publish applications using your own domain name](application-proxy-configure-custom-domain.md)
* [Enable single-sign on](application-proxy-configure-single-sign-on-with-kcd.md)
* [Troubleshoot issues you're having with Application Proxy](../active-directory-application-proxy-troubleshoot.md)


