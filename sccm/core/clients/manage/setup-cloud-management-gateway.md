---
title: "Set up Cloud Management Gateway | Microsoft Docs"
description: ""
author: robstackmsft
ms.author: robstack
manager: angrobe
ms.date: 05/01/2017
ms.topic: article
ms.prod: configuration-manager
ms.service:
ms.technology:
  - configmgr-client
ms.assetid: e0ec7d66-1502-4b31-85bb-94996b1bc66f
---

# Set up cloud management gateway for Configuration Manager

*Applies to: System Center Configuration Manager (Current Branch)*

Beginning in version 1610, the process for setting up cloud management gateway in Configuration Manager includes the following steps:

## Step 1: Configure required certificates

## Option 1 (preferred) - Use the server authentication certificate from a public and globally trusted certificate provider (like VeriSign)

When you use this method, clients will automatically trust the certificate, and you do not need to create a custom SSL certificate yourself.

1. Create a canonical name record (CNAME) in your organization’s public domain name service (DNS) to create an alias for the cloud management gateway service to a friendly name that will be used in the public certificate.
For example, Contoso names their cloud management gateway service **GraniteFalls** which in Azure will be **GraniteFalls.CloudApp.Net**. In Contoso’s public DNS contoso.com namespace, the DNS administrator creates a new CNAME record for **GraniteFalls.Contoso.com** for the real host name , **GraniteFalls.CloudApp.net**.
2. Next request a server authentication certificate from a public provider using the Common Name (CN) of the CNAME alias.
For example, Contoso uses **GraniteFalls.Contoso.com** for the certificate CN.
3. Create the cloud management gateway service in the Configuration Manager console using this certificate.
	- On the **Settings** page of the Create Cloud Management Gateway Wizard when you add the server certificate for this cloud service (from **Certificate file**), the wizard will extract the hostname from the certificate CN as the service name, and then append that to **cloudapp.net** (or **usgovcloudapp.net** for the Azure US Government cloud) as the Service FQDN to create the service in Azure.
For example, when creating the cloud management gateway at Contoso, the hostname **GraniteFalls** is extracted from the certificate CN, so that the actual service in Azure is created as **GraniteFalls.CloudApp.net**.

### Option 2 - Create a custom SSL certificate for cloud management gateway in the same way as for a cloud-based distribution point

You can create a custom SSL certificate for cloud management gateway in the same way you would do it for a cloud-based distribution point. Follow the instructions for [Deploying the Service Certificate for Cloud-Based Distribution Points](/sccm/core/plan-design/network/example-deployment-of-pki-certificates) but do the following things differently:

- When setting up the new certificate template, give **Read **and **Enroll** permissions to the security group that you set up for Configuration Manager servers.
- When requesting the custom web server certificate, provide an FQDN for the certificate's common name that ends in **cloudapp.net** for using cloud management gateway on Azure public cloud or **usgovcloudapp.net** for the Azure government cloud.


## Step 2: Export the client certificate's root

The easiest way to get export the root of the client certificates used on the network, is to open a client certificate on one of the domain-joined machines that has one and copy it.

> [!NOTE] 
>
> Client certificates are required on any computer you want to manage with cloud management gateway and on the site system server hosting the cloud management gateway connector point. If you need to add a client certificate to any of these machines, see [Deploying the Client Certificate for Windows Computers](/sccm/core/plan-design/network/example-deployment-of-pki-certificates#BKMK_client2008_cm2012).

1.  In the Run window, type **mmc** and press Return.

2.  From the File menu choose **Add/Remove Snap-in...**.

3.  In the Add or Remove Snap-ins dialog box, choose **Certificates** > **Add &gt;** > **Computer account** > **Next** > **Local computer** > **Finish**. 

4.  Go to **Certificates** &gt; **Personal** &gt; **Certificates**.

5.  Double-click the certificate for client authentication on the computer, choose the Certification Path tab, and double-click the root authority (at the top of the path).

6.  On the Details tab, choose **Copy to File...**.

7.  Complete the Certificate Export Wizard using the default certificate format. Make note of the name and location of the root certificate you create. You will need it to configure cloud management gateway in a [later step](#step-4-set-up-cloud-management-gateway).

## Step 3: Upload the management certificate to Azure

An Azure management certificate is required for Configuration Manager to access the Azure API and configure cloud management gateway. For more information and instructions for how to upload a management certificate, see the following articles in the Azure documentation:

- [Certificates overview for Azure Cloud Services](https://azure.microsoft.com/documentation/articles/cloud-services-certs-create/)

- [Upload an Azure Management API Management Certificate](https://azure.microsoft.com/documentation/articles/azure-api-management-certs/)

>[!IMPORTANT]
>Make sure to copy the subscription ID associated with the management certificate. You will need it for configuring cloud management gateway in the Configuration Manager console in the [next step](#step-4-set-up-cloud-management-gateway).

### Subordinate CA certificates and Azure

If your certificate is issued by a subordinate CA (subCA), and your enterprise PKI infrastructure is not on the Internet, use this procedure to upload the certificate to Azure. 

1. In your Azure portal, after setting up a cloud management gateway, locate the cloud management gateway service and go to the **Certificate** tab. Upload your subCA certificate(s) there. If you have more than one subCA cert, you need to upload all of them. 
2. Once the certificate is uploaded, record its thumbprint. 
3. Add the thumbprint to site database using this script:
	
```

	DIM serviceCName
	DIM subCAThumbprints

	' Verify arguments
	IF WScript.Arguments.Count <> 2 THEN
	WScript.StdOut.WriteLine "Usage: CScript UpdateSubCAThumbprints.vbs <ServiceCName> <SubCA cert thumbprints, separated by ;>"
	WScript.Quit 1
	END IF

	'Get arguments
	serviceCName = WScript.Arguments.Item(0)
	subCAThumbprints = WScript.Arguments.Item(1)

	'Find SMS Provider
	WScript.StdOut.WriteLine "Searching for SMS Provider for local site..."
	SET objSMSNamespace = GetObject("winmgmts:{impersonationLevel=impersonate}!\\.\root\sms")
	SET results = objSMSNamespace.ExecQuery("SELECT * From SMS_ProviderLocation WHERE ProviderForLocalSite = true")

	'Process the results
	FOR EACH var in results
	siteCode = var.SiteCode
	NEXT

	IF siteCode = "" THEN
	WScript.StdOut.WriteLine "Failed to locate SMS provider."
	WScript.Quit 1
	END IF

	WScript.StdOut.WriteLine "SiteCode = " & siteCode 

	' Connect to the SMS namespace
	SET objWMIService = GetObject("winmgmts:{impersonationLevel=impersonate}!\\.\root\sms\site_" & siteCode)

	'Get instance of SMS_AzureService
	DIM query
	query = "SELECT * From SMS_AzureService WHERE ServiceType = 'CloudProxyService' AND ServiceCName = '" & serviceCName & "'"
	WScript.StdOut.WriteLine "Run WQL query: " &  query
	SET objInstances = objWMIService.ExecQuery(query)

	IF IsNull(objInstances) OR (objInstances.Count = 0) THEN
	WScript.StdOut.WriteLine "Failed to get Azure_Service instance."
	WScript.Quit 1
	END IF

	FOR EACH var IN objInstances
	SET azService = var
	NEXT

	WScript.StdOut.WriteLine "Update [SubCACertThumbprint] to " & subCAThumbprints

	'Update SubCA cert thumbprints
	azService.Properties_.item("SubCACertThumbprint") = subCAThumbprints

	'Save data back to provider
	azService.Put_

	WScript.StdOut.WriteLine "[SubCACertThumbprint] is updated successfully."
```


## Step 4: Set up cloud management gateway

>[!NOTE]
>In version 1610, cloud management gateway is a pre-release feature. To enable it, see [Use pre-release features from updates](/sccm/core/servers/manage/install-in-console-updates#bkmk_prerelease).

1. In the Configuration Manager console, go to **Administration** > **Cloud Services** > **Cloud Management Gateway**.
2. Choose **Create Cloud Management Gateway**.

3. In Create Cloud Management Gateway Wizard, enter your Azure subscription ID (copied from the Azure management portal), and browse to the certificate file you used to upload as an Azure management certificate. Choose **Next** and then wait a few moments for the console to connect to Azure.

4. Fill out the additional details in the wizard:

    - Specify the name for the service which will run in Azure. Service name must be alphanumeric characters only and 3-24 characters in length.

    - Choose the Azure region you want the service to run in.

    - Specify the number of virtual machines you want to use for the service. The default is 1, but you can run up to 16 virtual machines for the service.

    >[!NOTE]
    >If you use an internet proxy for the cloud management gateway connection point, you need to increase the number of ports on the proxy by the number of virtual machines you use, starting at port 10124.

    - Specify the private key (.pfx file) that you exported from the custom SSL certificate.

    - Specify the root certificate exported from the client certificate.

    -   Specify the same service name FQDN that you used when you created the new certificate template. You must specify the one of the following suffixes for the FQDN service name based on the Azure cloud you are using:

    Azure cloud | FQDN prefix
    --------------|-------------
    Public (commercial) cloud | .cloudapp.net    
    Government cloud | .usgovcloudapp.net

  - Clear the box next to **Verify Client Certificate Revocation** (unless you're publicly publishing your CRL information).

  - Choose **Next** when you're done.

5. If you want to monitor cloud management gateway traffic with a 14-day threshold, choose the check box to turn on the threshold alert. Then, specify the threshold, and the percentage at which to raise the different alert levels. Choose **Next** when you're done.

6. Review the settings, and choose **Next**. Configuration Manager starts setting up the service. After you close the wizard it will take between 5 to 15 minutes to provision the service completely in Azure. Check the **Status** column for the newly setup cloud management gateway to determine when the service is ready.

## Step 5: Configure primary site for client certification authentication

1. In the Configuration Manager console, go to **Administration** > **Site Configuration** > **Sites**.

2. Select the primary site for the clients you want to manage through cloud management gateway, and choose **Properties**.

3. On the Client Computer Communications tab of the primary site property sheet, check **Use PKI client certificate (client authentication) when available**.

4. Make sure to clear **Clients check the certificate revocation list (CRL) for site systems**. This option would only be required if you were publicly publishing your CRL.


## Step 6: Add the cloud management gateway connector point

The cloud management gateway connector point is a new site system role for communicating with cloud management gateway. To add the cloud management gateway connector point, follow the instructions in [Add site system roles for System Center Configuration Manager](/sccm/core/servers/deploy/configure/add-site-system-roles).

## Step 7: Configure roles for cloud management gateway traffic

The final step in setting up cloud management gateway is to configure the site system roles to accept cloud management gateway traffic. Currently, only the management point, distribution point, and software update point roles are supported for cloud management gateway. You must configure each role separately.

1. In the Configuration Manager console, go to **Administration** > **Site Configuration** > **Servers and Site System Roles**.

2. Choose the site system server for the role you want to configure for cloud management gateway traffic.

3. Choose the role, and then choose **Properties**.

4. In the role Properties sheet, under Client Connections, choose **HTTP** or **HTTPS**, check the box next to **Allow Configuration Manager cloud management gateway traffic**, and then choose **OK**. Repeat these steps for the remaining roles.

## Step 8: Configure clients for cloud management gateway

After the cloud management gateway and site system roles are completely configured and running, clients will get the location of the cloud management gateway service automatically on the next location request. Clients must be on the corporate network to receive the location of the cloud management gateway service. The polling cycle for location requests is every 24 hours. If you don't want to wait for the normally scheduled location request, you can force the request by restarting the SMS Agent Host service (ccmexec.exe) on the computer.

With the location of the cloud management gateway service configured on the client, it can automatically determine whether it’s on the intranet or the Internet. If the client can contact the domain controller or the on-premises management point, it will use it for communicating with Configuration Manager, Otherwise, it will consider it’s on the Internet and use the location of the cloud management gateway service to communicate.

>[!NOTE]
> You can force the client to always use cloud management gateway regardless of whether it’s on the intranet or Internet. To do that, you set the following registry key on the client computer:\
>
> `HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\CCM\Security, ClientAlwaysOnInternet = 1`

To verify that clients can contact Configuration Manager, you can run the following PowerShell command on the client computer:

`gwmi -namespace root\ccm\locationservices -class SMS_ActiveMPCandidate`

This command displays the management points the client can contact including the cloud management gateway service.

## Next steps

[Monitor clients for cloud management gateway](monitor-clients-cloud-management-gateway.md)
