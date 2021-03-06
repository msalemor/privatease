# Deploying API Management and App Service Environment (ASE) with self-signed certificates

## 1.0 Who is this for

This is for a scenario where an organization may want to use API Management to consume services hosted on an ASE (APP Service environment) possibly during the dev/test phase of development.

<a href="solution-design.jpg"><img src="solution-design.jpg" alt="Screenshot"></a>

## 2.0 Requirements:

- Generate the rootCA and ASE ILB self-signed wildcard certificate
- Deploy the APIM Management in internal mode
  - Install the generated rootCA.cer in the root CA store
- Deploy the ASE in internal mode
  - Install the generated domain.pfx in as the ASE ILB certificate
  - Create an API app
- Configure the DNS to point to the API APP

### 2.1 Components


- <a href="#">Azure Virtual Network</a>  enables Azure resources to securely communicate with each other, the internet, and on-premises networks.
- <a href="#">Azure API Management</a>  helps organizations publish APIs to external, partner, and internal developers to unlock the potential of their data and services.
- <a href="#">Azure Application Service Environment (ASE)</a>  host isolated workloads with the same capabilities as App Services.

## 3.0 Installation Instructions

### 3.1 Generate the rootCA and ASE ILB self-signed wildcard certificate

Run this Powershell script to generate the rootCA certificate and the ASE ILB self-signed wildcard certificate.

```powershell
# Set the variables
$domain = "mycontosoase.com"
$secret = "StrongP@ssword"
$pfxPassword = ConvertTo-SecureString -String $secret -Force -AsPlainText
$certStore = "cert:\CurrentUser\My"

# Generate a rootCA using the domain name
$rootCertAse = New-SelfSignedCertificate -Type Custom -KeySpec Signature `
-Subject "CN=$domain" -KeyExportPolicy Exportable `
-HashAlgorithm sha256 -KeyLength 2048 `
-CertStoreLocation "$certStore" -KeyUsageProperty Sign -KeyUsage CertSign

# Export the rootCA's public key as a .cer
$rootCertAseThumbprint = $certStore + "\" + $rootCertAse.Thumbprint
Export-Certificate -Cert $rootCertAseThumbprint -FilePath "c:\certs\rootCA-$domain.cer"

# Generate a self-signed certificate for the ASE signed by the rootCA
$certAse = New-SelfSignedCertificate `
-HashAlgorithm sha256 -KeyLength 2048 `
-Signer $rootCert `
-dnsname "*.$domain","*.scm.$domain" `
-CertStoreLocation "$certStore"

$certAseThumbprint = $certStore + "\" + $certAse.Thumbprint
Export-PfxCertificate -cert $certAseThumbprint -FilePath "c:\certs\$domain.pfx" -Password $pfxPassword
```

### 3.2 Deploy API Management in internal mode

Run this Powershell script to create an API Manage instance in Azure:

```powershell
# Connect to Azure or log into a cloude shell and skip connecting to azure
Connect-AzAccount

# Create a resouce group
$resGroupName = "apim-appGw-RG" # resource group name
$location = "West US"           # Azure region
New-AzResourceGroup -Name $resGroupName -Location $location

# Create a virtual network
$apimsubnet = New-AzVirtualNetworkSubnetConfig -Name "apim02" -AddressPrefix "10.0.1.0/24"
$vnet = New-AzVirtualNetwork -Name "appgwvnet" -ResourceGroupName $resGroupName -Location $location -AddressPrefix "10.0.0.0/16" -Subnet $apimsubnet

# Create the API inside the vnet
$apimsubnetdata = $vnet.Subnets[1]
$apimVirtualNetwork = New-AzApiManagementVirtualNetwork -SubnetResourceId $apimsubnetdata.Id
$apimServiceName = "mycontosoapim"       # API Management service instance name
$apimOrganization = "Contoso"         # organization name
$apimAdminEmail = "admin@contoso.com" # administrator's email address
$apimService = New-AzApiManagement -ResourceGroupName $resGroupName -Location $location -Name $apimServiceName -Organization $apimOrganization -AdminEmail $apimAdminEmail -VirtualNetwork $apimVirtualNetwork -VpnType "Internal" -Sku "Developer"

```

Once the ASE is created, deploy the rootCA-mycontoso.com.cer to the root CA store. This is allow encrypted communication fro the APIM to the ASE.

INSERT IMAGE

### 3.3 Deploy the ASE in internal mode

In internal mode the ASE gets assigned an private IP from the subnet where it was deployed. To deploy an ASE:

- Log into the Azure Portal
- Create a new resource of type App Service Environment V2
- Select or create a resource group
- Select internal for the mode 
- Enter: mycontosoase.com for the domain.

<a href="create-internal-ase.jpg"><img src="create-internal-ase.jpg" width="75%" alt="Screenshot"></a>


#### 3.3.1 Update the ASE ILB Certificate

Once the ASE is created, update the ILB certificate with the mycontoso.com.pfx certificate. Note that this step can take up to one hour.

#### 3.3.2 Deploy an API in the ASE

Once the ILB certificate has been deploy, create a new service plan and API APP in the ASE. If this app is called for example "api1", it will receive a URI of https://api1.mycontosoase.com".

INSERT IMAGE

### 3.4 DNS Configuration

Once API Management and ASE have been deployed, configure the DNS so that the APIs point to the ASE's ILB IP, for example:

```
Name                          Type   Value
------------------------------------------------------------
ilbase.mycontosoase.com.      A      10.0.0.5
api1.mycontosoase.com.        CNAME  ilbase.mycontosoase.com.
```

Where 10.0.0.1 is the ASE's ILB IP.

### 3.5 Configure API Management

Follow the inscructions to create an API defintion using the ASE API URI defined in the step above.

## 4.0 References

- Integrate API Management in an internal VNET with Application Gateway
  - https://docs.microsoft.com/en-us/azure/api-management/api-management-howto-integrate-internal-vnet-appgateway
- Create and use an internal load balancer with an App Service Environment
  - https://docs.microsoft.com/en-us/azure/app-service/environment/create-ilb-ase
- Import and publish an API in API Management
  - https://docs.microsoft.com/en-us/azure/api-management/import-and-publish
