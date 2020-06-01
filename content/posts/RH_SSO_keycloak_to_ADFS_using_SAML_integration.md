---
date: 2020-04-11T18:00:00
description: "Implement a basic authentication combination configuration by identity brokering ADFS with RedHat SSO (Keycloak)"
#featured_image: "/images/keycloak_logo.png"
tags: ["ADFS","SAML"]
# categories: "POC"
title: "Integrating RH-SSO (Keycloak) to ADFS"
comment : false
---



This article aims to present the steps required to create the minimum necessary infrastructure, to gain familiarity with, and test Red Hat Single Sign-On (SSO) and its integration with Microsoft Active Federation Services (ADFS) via the SAML protocol.

Most businesses usually have a large Windows estate and therefore use Active Directory (AD) as it provides a single source of user management in the organisation.  Many organizations often incorporate additional authentication programs and protocols in tandem with AD, including RH-SSO for example.

This is not a reference architecture, nor does it illustrate the best-practices for the technologies shown. To achieve a more production style implementation, this article showcases a high-level combination of the official Microsoft ADFS documentation and the RedHat SSO documentation.

The goal of this article is to simply illustrate the bare minimum configurations required to become familiar with the capabilities of the products, an example of an integration method, and how that integration can be validated and tested. At the end of the article, the following will have configured:

* Windows Server 2016 server with the following roles: DC, ADFS, IIS
* Red Hat Enterprise Linux server running RedHat Single Sign-On
* Trust between SSO and ADFS
* User authentication against AD via SSO


# Overview

### Active Directory as an Identity Provider
ADFS is the web module that provides endpoints to allow the use of security tokens - either OpenID Connect (OIDC) or SAML Assertions with an Active Directory Server.ADFS is typically encountered as being the bolt-on webserver to AD on-premise, in which case it is likely to be fairly old. More commonly these days, the Azure version of ADFS is used, which is more opinionated and consequently, perhaps easier to work with.

Since every AD system administrator will have a differing view on how they should structure their AD database, it is not practical to provide one-size-fits-all guidance. However, this article will at least give you, the reader, some pointers around the basic steps on how to go about setting up integrations with ADFS.



### Integration
The Unified Modeling Language (UML) diagram below shows at a high level what the authentication steps are for this article using ADFS as an authentication provider for RedHat SSO using the SAML protocol. To be clear, this is just one example of Identity Brokering; SAML is highly flexible, as is RH-SSO.

![ADFS SSO Integration](/images/ADFS_SSO_SAML.png#center)

Flow Description:
1. The user clicks on the SAML button on the RH-SSO form.
2. This sends a redirect to the browser along with a SAML request for Auth.
3. The browser sends an HTTP GET to the ADFS server passing the parameters for the Auth.
4. ADFS returns a login form requesting the user to login.
5. The user completes the login form and then submits this form, which in-turn HTTP POSTs the credentials back to the ADFS server.
6. ADFS checks the credentials and, when legitimate, posts a SAML response with the mapped claims to the RH-SSO server. (BTW - the endpoint POSTed to was configured originally as a part of the setup, this came from the metadata exported from RH-SSO.)
7. On receipt of the SAML Response, RH-SSO provides the user with the option to update their details. On submission, the credentials are added to RH-SSO’s user database.
8. END.


### Topology
The below diagram is a simple architectural example in terms of how ADFS and SSO can be configured and integrated. The diagram illustrates the basic topology at a high-level for how the services are aware of each other; which is basically through the act of exchanging metadata files.

![ADFS SSO Topology](/images/ADFS_SSO.png#center)


* **FederationMetadata.xml** - This file contains information about your federation service that is used to create trusts, identify token-signing certificates, and many other things. So it needs to be publicly available so that other parties can access and consume it. This file is generated first.
* **IdentityProviderMetadata.xml**  - This file is the metadata produced and downloaded from RedHat SSO which contains the SAML service provider entity descriptor which you can use to import into ADFS. This file is generated second.

### Infrastructure Architecture

To minimise the infrastructure footprint of ADFS for the build, I’ve opted to provision a Windows Server 2016 VM, acting as the full Windows estate and a RHEL 8 VM, which will run SSO. In a more realistic scenario, the Domain Controller (DC), Federation server ADFS, and Web Server (IIS) services would be split and made highly available. However, as mentioned this article only illustrates how to quickly build a lab environment and show the integration between the two products.

| Purpose           | Role    | Host  | OS              |
|-------------------|---------|-------|-----------------|
| Domain Controller | DC, DNS | ad01  | Win Server 2016 |
| Federation server | ADFS    | ad01  | Win Server 2016 |
| WebServer         | IIS     | ad01  | Win Server 2016 |
| RH-SSO            | JBOSS   | sso01 | RHEL8           |

### Approach
All setup steps in this document use CLI commands as much as is possible, Powershell in the case of Windows hosts, or Bash for Linux hosts. Tasks are presented in this fashion so the reader can understand the manual steps involved in the process. In the real world, automation should be used to reduce deployment and configuration time and increase consistency, repeatability, and reliability. This article assumes that the machines are set up and are on the same network and can communicate with each other freely.

### Installation Media
In order to work through these examples you will need:

* Red Hat Enterprise Linux 8 installation media or virtual machine templates
* Microsoft Windows Server 2016 installation media or virtual machine templates
* Credentials for Red Hat Subscription Manager (RHSM)
* RedHat Single Sign-On installation media

Windows Server 2016 evaluations:
> https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2016

Red Hat Enterprise Linux Developer instances (You can instead use CentOS):
> https://developers.redhat.com/products/rhel/download/

RedHat SSO media (You can instead use keycloak):
> https://access.redhat.com/jbossnetwork/restricted/listSoftware.html?downloadType=distributions&product=core.service.rhsso


# Step 1: Configure the Windows Domain Controller (DC)

### Install Active Directory Domain Services (AD DS)

In this example, we will model a single organisation: Contoso (contoso.com). The following Powershell will install the AD Domain Services Windows feature and its dependencies, invoke the deployment module, then configure the target system as an AD DC (and configure DNS for the domain).

```text
PS> Rename-Computer -NewName ad01
```

```text
PS> Get-windowsfeature
PS> Install-WindowsFeature -name NET-Framework-Core
PS> Install-windowsfeature -name AD-Domain-Services -IncludeManagementTools
```

```text
PS> Import-Module ADDSDeployment
PS> Install-ADDSForest `
-CreateDnsDelegation:$false `
-DatabasePath "C:\Windows\NTDS" `
-DomainMode "WinThreshold" `
-DomainName "contoso.com" `
-DomainNetbiosName "contoso" `
-ForestMode "WinThreshold" `
-InstallDns:$true `
-LogPath "C:\Windows\NTDS" `
-NoRebootOnCompletion:$false `
-SysvolPath "C:\Windows\SYSVOL" `
-Force:$true `
-SafeModeAdministratorPassword (ConvertTo-SecureString –String 'Pa33w0rd!' -AsPlainText -Force)
```

### Validate Configuration
```text
PS> Get-Service adws,kdc,netlogon,dns
PS> Get-SmbShare
PS> dcdiag /q
PS> Resolve-DnsName ad01.contoso.com
PS> Resolve-DnsName contoso.com
```

### Configure Users and Groups
Active Directory Security Groups can group domain users with similar roles, departments, organisational responsibilities, or to reflect other organisational concerns. Permissions can then be assigned at the group level, reducing the management overhead as users join, change role or department, or leave. The following Powershell is an example that will create domain security groups, domain users, and assign those users to the groups.

### Create Directories
```text
PS> New-ADGroup -Name "Officers" -SamAccountName Officers -GroupCategory Security -GroupScope Global -DisplayName "Bridge Officers" -Path "CN=Users,DC=contoso,DC=Com" -Description "Members of Bridge Officers"
PS> New-ADGroup -Name "Engineers" -SamAccountName Engineers -GroupCategory Security -GroupScope Global -DisplayName "Engineering Crew" -Path "CN=Users,DC=contoso,DC=Com" -Description "Members of Engineering Crew"
```

### Create Users
```text
PS> $Attributes = @{
   Enabled = $true
   ChangePasswordAtLogon = $false
   UserPrincipalName = "dallas@contoso.com"
   Name = "dallas"
   GivenName = "Captain"
   Surname = "Dallas"
   DisplayName = "Captain Dallas"
   Office = "Bridge"
   AccountPassword = "Thatfigures." | ConvertTo-SecureString -AsPlainText -Force
}
PS> New-ADUser @Attributes
```

```text
PS> $Attributes = @{
   Enabled = $true
   ChangePasswordAtLogon = $false
   UserPrincipalName = "kane@contoso.com"
   Name = "kane"
   GivenName = "XO"
   Surname = "Kane"
   DisplayName = "XO Kane"
   Office = "Bridge"
   AccountPassword = "Sillyquestion?" | ConvertTo-SecureString -AsPlainText -Force
}
PS> New-ADUser @Attributes
```

```text
PS> $Attributes = @{
   Enabled = $true
   ChangePasswordAtLogon = $false
   UserPrincipalName = "parker@contoso.com"
   Name = "parker"
   GivenName = "Chief"
   Surname = "Parker"
   DisplayName = "Chief Parker"
   Office = "Engineering"
   AccountPassword = "Howyadoin?" | ConvertTo-SecureString -AsPlainText -Force
}

PS> New-ADUser @Attributes
```
```text
PS> New-ADUser -Name "John Smith" -GivenName "John" -Surname "Smith" -SamAccountName "jsmith" -UserPrincipalName "jsmith@contoso.com" -Path "CN=Users,DC=contoso,DC=Com" -AccountPassword(Read-Host -AsSecureString "Input Password") -Enabled $true
```

### Add Users to Security Groups
```text
PS> Add-ADGroupMember -Identity Officers -Members dallas, kane
PS> Add-ADGroupMember -Identity Engineers -Members parker
```

### Validate
```text
PS> net user kane
PS> net user parker
PS> net user dallas
```

### Create a GMSA account
This account can also be used during the Install-AdfsFarm process instead of giving credentials of an admin if preferred.
```text
Add-KdsRootKey -EffectiveTime (Get-Date).AddHours(-10)
New-ADServiceAccount FsGmsa -DNSHostName ad01.contoso.com -ServicePrincipalNames http/ad01.contoso.com
```







# Step 2: Configure the Federation Server (ADFS)

### Install ADFS
The following Powershell will install the ADFS feature and its dependencies, invoke the deployment module, and then set up a certificate. For ease of use with this being a lab environment, we created a wildcard certificate rather than the suggested method of creating specific common names.
```text
PS> PS> Install-windowsfeature adfs-federation –IncludeManagementTools
PS> Import-Module ADFS

PS> add-windowsfeature adcs-cert-authority -IncludeManagementTools
PS> Install-AdcsCertificationAuthority -CAType EnterpriseRootCa
PS> Install-PackageProvider nuget -force
PS> Install-Module -Name PSPKI -Force
PS> Import-Module -Name PSPKI

PS> New-SelfSignedCertificate -certstorelocation cert:\localmachine\Root -dnsname *.contoso.com
PS> Get-ChildItem -path cert:\LocalMachine\My
```

### Configure ADFS
When running the below Powershell, a credentials prompt appears. The format is domain\user e.g. ‘contoso.com\Administrator’. As for the below hashes, please replace them with the cert thumbprint for the wildcard cert which can be viewed again by the Get-ChildItem command.
```text
PS> $fscredential = Get-Credential
PS> Get-ChildItem -path cert:\LocalMachine\My

PS> Install-AdfsFarm -CertificateThumbprint ################################ -FederationServiceName ad01.contoso.com -ServiceAccountCredential $fscredential

PS> Install-AdfsFarm -CertificateThumbprint ################################ -FederationServiceName ad01.contoso.com -ServiceAccountCredential $fscredential -OverwriteConfiguration

PS> Set-AdfsProperties -EnableIdPInitiatedSignonPage $true
```
Note: If you encounter an error and want to rerun use the “-OverwriteConfiguration” flag

### Configure Device Registration Service
For the prompt enter the domain and GMSA account setup earlier e.g. in this case the below
```text
PS> Initialize-ADDeviceRegistration
contoso.com\fsgmsa$
PS> Enable-AdfsDeviceRegistration
```

Note: in Win Server 2016, the command 'Enable-AdfsDeviceRegistration' is obsolete. The Cmdlet has been deprecated. Install-AdfsFarm and Add-AdfsFarmNode perform the required tasks to enable Device Registration Service. However, this is worth knowing if the reader is using an older version of Windows server.

```text
PS> Set-AdfsGlobalAuthenticationPolicy –DeviceAuthenticationEnabled $true
```

### DNS
Add the required DNS records associated for your environment.
```text
PS> Get-DnsServerResourceRecord -ZoneName contoso.com

PS> Add-DnsServerResourceRecordA -Name ad01 -ZoneName contoso.com -IPv4Address ###.###.##.###
PS> Get-DnsServerResourceRecord -ZoneName contoso.com -RRType A

PS> Add-DnsServerResourceRecordCName -ZoneName contoso.com -HostNameAlias "ad01.contoso.com" -Name "enterpriseregistration"
PS> Get-DnsServerResourceRecord -ZoneName contoso.com -RRType CName
```


# Step 3: Configure the Web Server (IIS)
Install the Web Server role and Windows Identity Foundation
The following Powershell will install the IIS feature and its dependencies as well as the dot net framework.
```text
PS> Install-WindowsFeature Web-Server -IncludeManagementTools -IncludeAllSubFeature -ComputerName ad01 -WhatIf
PS> Install-WindowsFeature Windows-Identity-Foundation -IncludeManagementTools
PS> Install-WindowsFeature Net-Framework-Core
```

### Validate ADFS
The below web address will validate if ADFS is working.
```text
PS> Start-Process "https://ad01.contoso.com/adfs/fs/federationserverservice.asmx”
PS> Start-Process “https://ad01.contoso.com/adfs/ls/idpinitiatedsignon.htm”
PS> Start-Process “https://ad01.contoso.com/FederationMetadata/2007-06/FederationMetadata.xml”
```







# Step 4: RH-SSO Installation

### RedHat SSO installation

The following Shell will install Java8, unpack SSO in a relevant directory and setup an administrator user. In this example that user is named ‘sysadmin’.

```text
# sudo yum install jdk-8

$ mv rh-sso-7.3.0.GA.zip /opt
$ cd /opt
$ jar xf rh-sso-7.3.0.GA.zip
$ chmod -R 770 rh-sso-7.3
$ cd rh-sso-7.3/bin
$ ./add-user-keycloak.sh -u sysadmin
```

Note: amended standalone.xml to bind too all available addresses for the exercise for ease of use.

```text
$ cd /opt/rh-sso-7.3/standalone/configuration
$ sed -i 's/192.168.0.1/0.0.0.0/g' standalone.xml
$ ./standalone.sh
```

A logon screen for SSO should now be available:

> `http://<HOST>:8080/auth/`

Now using the WebConsole create a realm named ‘demo’. If you need assistance with this follow the SSO getting started guide.

> https://access.redhat.com/documentation/en-us/red_hat_single_sign-on/7.3/html/getting_started_guide/index

### SSL setup
The Shell below checks connectivity to the ADFS server, exports that certificate, and then imports that server's certificate into a system truststore.
```text
$ cd /opt/rh-sso-7.3
$ wget https://ad01.contoso.com/federationmetadata/2007-06/federationmetadata.xml
$ openssl s_client -connect ad01.contoso.com:443 2>/dev/null </dev/null |  sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' >> ad01.cer
$ cp ad01.cer /usr/share/pki/ca-trust-source/anchors/
$ update-ca-trust
$ trust list
$ wget https://ad01.contoso.com/federationmetadata/2007-06/federationmetadata.xml
```

### SSL - Outgoing HTTP Requests
The Shell below creates a java trust store for JBOSS to use and then imports the certificate of the ADFS server into it.
```text
$ cd /opt/rh-sso-7.3
$ keytool -import -alias ad01.contoso.com -keystore truststore.jks -file ad01.cer
```

Add the below block for the truststore configuration to the standalone.xml configuration file (near the bottom under the spi sections).
```xml
 <spi name="truststore">
     <provider name="file" enabled="true">
         <properties>
             <property name="file" value="/opt/rh-sso-7.3/truststore.jks"/>
             <property name="password" value="test77"/>
             <property name="hostname-verification-policy" value="ANY"/>
             <property name="disabled" value="false"/>
         </properties>
     </provider>
 </spi>
```

After amending the standalone.xml file, restart JBOSS for the changes to pickup.

### Enabling SSL/HTTPS for the Keycloak Server (Inbound)
The Shell below creates a java trust store with a self-signed certificate.
```text
$ cd /opt/rh-sso-7.3
$ keytool -genkey -alias localhost -keyalg RSA -keystore keycloak.jks -validity 10950
Enter keystore password:  test77
Re-enter new password: test77
$ keytool -importkeystore -srckeystore keycloak.jks -destkeystore keycloak.jks -deststoretype pkcs12
$ ./bin/kcadm.sh config truststore --trustpass test77 /opt/rh-sso-7.3/keycloak.jks
$ cp keycloak.jks standalone/configuration
```
The below shell will invoke the JBOSS-CLI, where we execute some JBOSS server configuration to add the keystore information and enable extra logging.
```text
$ cd bin
$ ./jboss-cli.sh
```
```text
/connect

/core-service=management/security-realm=UndertowRealm:add()

/core-service=management/security-realm=UndertowRealm/server-identity=ssl:add(keystore-path=keycloak.jks, keystore-relative-to=jboss.server.config.dir, keystore-password=test77)

/subsystem=undertow/server=default-server/https-listener=https:write-attribute(name=security-realm, value=UndertowRealm)

/reload

/subsystem=logging/logger=org.keycloak.saml:add(level=DEBUG)

/subsystem=logging/logger=org.keycloak.broker.saml:add(level=DEBUG)
```










# Step 5: SSO - ADFS integration

In the RedHat SSO Web GUI configure an Identity Provider:
* Click Add Provider (SAML v2)
* At the bottom of the page select ‘Import from URL’ and enter the ADFS server Federation metadata URL e.g.: `https://ad01.contoso.com/federationmetadata/2007-06/federationmetadata.xml`
* Set NameID Policy Format to email
* Save

![SSO ADFS Integration](/images/sso1.png#center)<br>
<br>

![SSO ADFS Integration](/images/sso2.png#center)<br>
<br>

![SSO ADFS Integration](/images/sso3.png#center)<br>
<br>

![SSO ADFS Integration](/images/sso4.png#center)<br>
<br>

* Export the SAML endpoint metadata

This can then be saved and copied to the Windows Server
Or alternatively the same metadata is available from the URL:

> `https://<HOST>:<PORT>/auth/realms/<Realm>/broker/<IdentityProvider>/endpoint/descriptor`

Note: Be sure that the export metadata is using the HTTPS SSL addresses. If not, that is okay however you need to edit these addresses for HTTPS and amend the port, and you will have to do the file import on ADFS and not the URL import.

![SSO ADFS Integration](/images/sso5.png#center)

Alternatively, the following URL will show the descriptor metadata, which can be used instead for the ADFS server configuration of SSO.

> `https://<HOST>:<PORT>/auth/realms/<Realm>/broker/<IdentityProvider>/endpoint/descriptor`









# Step 6: ADFS - SSO integration
### Adding Relying Party Trusts
The following Powershell will import the metadata for the configuration details of the SSO server either from the metadata XML file previously downloaded or directly from the URL.

From a File:
```text
PS> Add-AdfsRelyingPartyTrust -Name "SSO" -MetadataFile "c:\metadatafile.xml" -AccessControlPolicyName "Permit Everyone"
```
Specify the metadata xml file as it allows the xml file to be modified, e.g. if the output is HTTP endpoints and not HTTPS.

From a URL:
```text
PS> Add-AdfsRelyingPartyTrust -Name "SSO" -MetadataURL "https://<HOST>:<PORT>/auth/realms/<REALM_NAME>/broker/saml/endpoint/descriptor" -AccessControlPolicyName "Permit Everyone"
```
Provide the URL for SSO’s metadata like we mentiomed earlier:

> `https://<HOST>:8443/auth/realms/<REALM_NAME>/broker/saml/endpoint/descriptor`

Adding Relying Party Trusts Mapping Rules
Create a text document e.g. "C:\rules.txt"and place the following inside it:
```text
@RuleTemplate = "MapClaims"
@RuleName = "Name ID"
c:[Type == "http://schemas.microsoft.com/ws/2008/06/identity/claims/windowsaccountname"]
 => issue(Type = "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/nameidentifier", Issuer = c.Issuer, OriginalIssuer = c.OriginalIssuer, Value = c.Value, ValueType = c.ValueType, Properties["http://schemas.xmlsoap.org/ws/2005/05/identity/claimproperties/format"] = "urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress");

@RuleTemplate = "LdapClaims"
@RuleName = "Email, subject"
c:[Type == "http://schemas.microsoft.com/ws/2008/06/identity/claims/windowsaccountname", Issuer == "AD AUTHORITY"]
 => issue(store = "Active Directory", types = ("http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress", "http://schemas.microsoft.com/2012/12/certificatecontext/field/subjectname"), query = ";mail,sAMAccountName;{0}", param = c.Value);
```
Then run the below powershell command to add the rules
```text
PS> Set-AdfsRelyingPartyTrust -TargetName "Destination Relying Party Trust" -IssuanceTransformRulesFile "C:\rules.txt"
```

If you would like to add additional rules with the ADFS Management GUI, you can then export the rules again to text with:
```text
PS> (Get-AdfsRelyingPartyTrust -Name "saml").IssuanceTransformRules | Out-File "C:\rules.txt"
```

### Example of adding rules via GUI (optional)
The following is not needed if the previous powershell was run. However it does illustrate how to add relying party trust rules for mapping which is in fact easier initially to do via the GUI if you are unsure on exactly the mapping configuration. The two examples below show how the previous configuration was doen for mapping user ID and user attributes.

### Rule for Mapping user ID (GUI)
* In Relying Party Trusts select the newly created item and then in the right hand side Actions bar select ‘Edit Claim Issuance Policy’
* Select ‘Add Rule’
* For the ‘Claim rule template’ select ‘Transform an incoming claim’.
* Map the the below attributes:
* Click Finish to add the rule.

| Claim rule name         | Name ID              |
|-------------------------|----------------------|
| Incoming claim type     | Windows account name |
| Outgoing claim type     | Name ID              |
| Outgoing name ID format | Email                |

### Rule for Mapping the Attributes of the Standard User (GUI)
*  In Relying Party Trusts select the newly created item and then in the right hand side Actions bar select ‘Edit Claim Issuance Policy’
Select ‘Add Rule’
* For the ‘Claim rule template’ select ‘Send LDAP attributes as Claims’.
* Map the following attributes:
* Click Finish to add the rule.


| Claim rule name         | Name ID              |
|-------------------------|----------------------|
| E-Mail-Addresses        | E-Mail Address       |
| SAM-Account-Name        | Subject Name         |

Add other attributes if needed such as your LDAP attributes for surname and given name







# Step 7: Testing and Validation
In a web browser open your SSO logon page for the realm you setup:

> `https://<SSO_HOST>:8443/auth/realms/<REALM>/account`

Click SAML, and you will be redirected to your organization's page (ADFS).

![SSO ADFS Integration](/images/sso7.png#center)

Logon with an example users AD credentials to validate that AD domain users can authenticate against SSO with their domain credentials.

![SSO ADFS Integration](/images/sso8.png#center)

You will then be logged in and redirected back to SSO, in which you will now see your account details

![SSO ADFS Integration](/images/sso9.png#center)

Once a user has successfully logged on you can see a response like the below that will be found in the JBOSS server logs. Assuming that you have installed RH-SSO inline with the description above then the logs will be found at $JBOSS_HOME/standalone/logs/server.log

You may have to turn on the debug in order to see these messages. This is typically done via the following commands:

```text
$JBOSS_HOME/bin/jboss-cli.sh --connect
/subsystem=logging/logger=org.keycloak/:add(category=org.keycloak,level=TRACE)
```

```xml
<?xml version="1.0" encoding="UTF-8"?>samlp:Response ID="_b37dfe49-822b-46ec-8919-869ebfa63b2d" Version="2.0" IssueInstant="2020-02-20T12:26:54.794Z" Destination="https://192.168.56.102:8443/auth/realms/demo/broker/saml/endpoint" Consent="urn:oasis:names:tc:SAML:2.0:consent:unspecified" InResponseTo="ID_dd02e34f-40b2-4a2a-b972-87be50a8a9a7" xmlns:samlp="urn:oasis:names:tc:SAML:2.0:protocol">
<Issuer xmlns="urn:oasis:names:tc:SAML:2.0:assertion">http://ad01.contoso.com/adfs/services/trust</Issuer>
<samlp:Status>
  <samlp:StatusCode Value="urn:oasis:names:tc:SAML:2.0:status:Success" />
</samlp:Status>
<Assertion ID="_8f000dc3-31cf-48c0-80d9-bcccaa409e2e" IssueInstant="2020-02-20T12:26:54.764Z" Version="2.0" xmlns="urn:oasis:names:tc:SAML:2.0:assertion">
  <Issuer>http://ad01.contoso.com/adfs/services/trust</Issuer>
  <ds:Signature xmlns:ds="http://www.w3.org/2000/09/xmldsig#">
    <ds:SignedInfo>
      <ds:CanonicalizationMethod Algorithm="http://www.w3.org/2001/10/xml-exc-c14n#" />
      <ds:SignatureMethod Algorithm="http://www.w3.org/2001/04/xmldsig-more#rsa-sha256" />
      <ds:Reference URI="#_8f000dc3-31cf-48c0-80d9-bcccaa409e2e">
        <ds:Transforms>
          <ds:Transform Algorithm="http://www.w3.org/2000/09/xmldsig#enveloped-signature" />
          <ds:Transform Algorithm="http://www.w3.org/2001/10/xml-exc-c14n#" />
        </ds:Transforms>
        <ds:DigestMethod Algorithm="http://www.w3.org/2001/04/xmlenc#sha256" />
        <ds:DigestValue>VnPFSxcb/KOo+YS3Lf8h7UPgTWhlAt97RdkkFD7VyXw=</ds:DigestValue>
      </ds:Reference>
    </ds:SignedInfo>
    <ds:SignatureValue>5mX/hgPKocmMx7hxo1eB3/xtOOmrWjxXn3qhdI041B1w0tDl4a8p/FPmVHc85BxkXLfbPYbfbB7lAsYbTNMVPtl5IzF2vsTxVuJBRRtwko39PYbPGeVhbCQMeazxUMQEAcYtD5JL4slofNYmq6biH0iYbNYgvxlh7LMmHsg2Es1GjY9DvtidbGYrtNvUSysSuuaGznZbtFWWpbDOuNubJpGTEvSYO9H2AAqx9cY9376bBF6ui/deQ/4mTFK5OY3nvYdi4hs4s/fN4qvdqKm9JbuBjV3tuFZ8WEJrwFApD05yqJRA4bZkjyjU1Y0C/zGp4RaFE026Uv20wdr1gX7W/g==</ds:SignatureValue>
    <KeyInfo xmlns="http://www.w3.org/2000/09/xmldsig#">
      <ds:X509Data>
        <ds:X509Certificate>MIIC3DCCAcSgAwIBAgIQJjwM5Bqr44xDPLzkb1zUEDANBgkqhkiG9w0BAQsFADAqMSgwJgYDVQQDEx9BREZTIFNpZ25pbmcgLSBhZDAxLmNvbnRvc28uY29tMB4XDTIwMDIxOTEwNDQyOFoXDTIxMDIxODEwNDQyOFowKjEoMCYGA1UEAxMfQURGUyBTaWduaW5nIC0gYWQwMS5jb250b3NvLmNvbTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAO6P4+vK085uyOqs+0++l4KAk6tF6x4RScYecCYwcaFg+VqaOI1dvrwaVMSLLi39ORUEVNIw+Ju5YbjCB3tVINliW+P1wnf7Y3k0QHEgXZTmJ19FbGjHySNDXSkd+4J242kiuYZfU1nQfCkCLCqXgSZwj5apnaV1cNgKxB9TE0cDl1FF3O1j2t7P+kTV+rEOZuIWy4UJfwGRQFBxTIB95pKJssY5vB/3MldXKOspdjXg6zvmpQ3Jjx50lbyxLGVlol/otPa7UAQtFzW1D7Bbl6V5GE06OSGcO9KI1dtKwKoiTC5B5FGXGAiYRBcLV7adzEANpVpwOTdzq6DSjFY8dC8CAwEAATANBgkqhkiG9w0BAQsFAAOCAQEAmFk1RNHAvj5VLzgBMVyA4NY3ruSuV9JMnPnmLabM/KXOiczWq+4sqLRCyB4gREm/gxkORqlAwrW0Rla3ZSpYRjvSIMmWL2bHZu52hrJMkxLQu1laNI1bRXalchcuWxkgaj2LMiT3sPogf6/Y8Ffizn6wyqOXyVLSJoFSEBHiuKoaKDlpz/rRd7bNDenIYyNr6mOA2rpAptG4EjSlAVX2YuholFF1kizdOmSCS+Y8k3/9fhgYYQ7hHZr5ObAOkjUucRO/OwZtghkUtMnrf+KCrLDsUTAwNCxbxtk1ACDjDfwRau9vq42YLqqAgAoitfdccnWOGdi9RmubHVAkXfSdlQ==</ds:X509Certificate>
      </ds:X509Data>
    </KeyInfo>
  </ds:Signature>
  <Subject>
    <NameID Format="urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress">contoso\Administrator</NameID>
    <SubjectConfirmation Method="urn:oasis:names:tc:SAML:2.0:cm:bearer">
      <SubjectConfirmationData InResponseTo="ID_dd02e34f-40b2-4a2a-b972-87be50a8a9a7" NotOnOrAfter="2020-02-20T12:31:54.801Z" Recipient="https://192.168.56.102:8443/auth/realms/demo/broker/saml/endpoint" />
    </SubjectConfirmation>
  </Subject>
  <Conditions NotBefore="2020-02-20T12:26:54.755Z" NotOnOrAfter="2020-02-20T13:26:54.755Z">
    <AudienceRestriction>
      <Audience>https://192.168.56.102:8443/auth/realms/demo</Audience>
    </AudienceRestriction>
  </Conditions>
  <AttributeStatement>
    <Attribute Name="http://schemas.microsoft.com/2012/12/certificatecontext/field/subjectname">
      <AttributeValue>Administrator</AttributeValue>
    </Attribute>
  </AttributeStatement>
  <AuthnStatement AuthnInstant="2020-02-20T12:26:54.643Z" SessionIndex="_8f000dc3-31cf-48c0-80d9-bcccaa409e2e">
    <AuthnContext>
      <AuthnContextClassRef>urn:oasis:names:tc:SAML:2.0:ac:classes:PasswordProtectedTransport</AuthnContextClassRef>
    </AuthnContext>
  </AuthnStatement>
</Assertion>
</samlp:Response>
```

Congratulations. You have successfully setup SSO integration with ADFS.
