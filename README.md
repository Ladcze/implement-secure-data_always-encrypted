# implement-secure-data (Always Encrypted)
Implementing Secure Data by setting up Always Encrypted - Azure Key Vault

---

## Introduction
This depicts a proof of concept application which is deployed using the Always Encrypted functionality provided by Azure's SQL Database. Relevant secrets and keys applicable in this instance will be stored in Azure Key Vault. 
An application based on .NET (C#) will be registered in Microsoft Entra ID to help enhance its overall security posture. 

To accomplish the above objectives, the proof of concept includes:
- Creating an Azure Key Vault and storing keys and secrets in the said vault.
- Creating a SQL Database and encrypting content of columns in database tables by using Always Encrypted.

---

## Services deployed
- These are the 3 primary services required for this project

![](/imgs/keyvault.jpeg)       ![](/imgs/AzureSQL.jpg)  ![](/imgs/azureVM.png) 

- Other services deployed include:   
StorageAccount   
SQLServer   
vNet   
NetworkInterface   
PublicIP   
NSG   

---

## Steps (in summary)
1. Deploy the relevant base infrastructure
2. Configure the Key Vault resource with a key and a secret
3. Configure an Azure SQL database and a data-driven application
4. Demonstrate the use of Azure Key Vault in encrypting the Azure SQL database

*** 
***

## 1. Deploy the base infrastructure from an ARM template.    
Deploy an Azure VM and an Azure SQL database.    

Sign in to the Azure portal using an account with the Owner or Contributor role in your subscription.
Using the deploy custom template option, the relevant baseline resources can be deployed.  

![See 01 deployTemplate.json](https://github.com/Ladcze/implement-secure-data_always-encrypted/blob/3305b02e4c12d29a2178225ab6bdea30529823bd/01%20deployTemplate.json)  
The above sequence will initiate the deployment of the Azure VM and Azure SQL Database required for this proof of concept.  
Ensure the following settings are configured: Resource group, Location, and Password.  
  

*** 
***

## 2. Configure the Key Vault resource with a key and a secret  
## 2.1 Create and configure a Key Vault ##
Create an Azure Key Vault resource and configure the Azure Key Vault permissions. 
* ### Create an Azure Key Vault resource
In the PowerShell session within the Cloud Shell pane, run the Powershell script to create an Azure Key Vault in the specified resource group. 
The key Vault name must be unique.  
![See 02a CreateKV.ps1](https://github.com/Ladcze/implement-secure-data_always-encrypted/blob/3305b02e4c12d29a2178225ab6bdea30529823bd/02a%20CreateKV.ps1) 

* ### Configure Vault permission(s)
In the Azure portal, go to rg-SecOps resource group
Go to key vault entry "a5w2cKV"
Navigate to the Access Policies and then click + Create. 
Specify the following value settings (leave all others with their default values): 
* Key permissions - check Select all (9) permissions 
* Key permissions/Cryptographic Operations - check "Sign" permission only  
* Secret permissions - check Select all (7) permissions 
* Certification permissions - check all (15) permissions 

Assign principal: click None selected, on the Principal blade, select your user account, and click Next 

Click Review + Create (to validate) then click Create  

<br>

## 2.2 Add a key to the Key Vault ## 
Add a key to the Key Vault and view information about the key. 
* In a PowerShell session within the Cloud Shell pane, run the Powershell script to add a software-protected key to the Key Vault:  
![See 02b KV.ps1](https://github.com/Ladcze/implement-secure-data_always-encrypted/blob/3305b02e4c12d29a2178225ab6bdea30529823bd/02b%20KV.ps1)   
Note: The name of the key is w2cLabKey 

* To Verify successful key creation run  
*<ins>Get-AZKeyVaultKey -VaultName $kv.VaultName<ins>* 

* To display the key identifier run  
*<ins>$key.key.kid<ins>*

<br>

## 2.3 Add a secret to the Key Vault ##
* In the PowerShell session within the Cloud Shell pane, run the following to create a variable with a secure string value:  
Code (# create a variable)  
*<ins>$secretvalue = ConvertTo-SecureString 'Billy-Dash33p@!' -AsPlainText -Force<ins>*

* Add a secret  
In the PowerShell session, run the following to add the secret to the vault:  
Code (# add a secret)  
*<ins>$secret = Set-AZKeyVaultSecret -VaultName $kv.VaultName -Name 'SQLPassword' -SecretValue $secretvalue<ins>*  

Note: The name of the secret is SQLPassword.  

* To verify the secret was created run code  
*<ins>Get-AZKeyVaultSecret -VaultName $kv.VaultName<ins>* 

*** 
***

## 3. Configure an Azure SQL database and a data-driven application  

## 3.1 Enable a client application to access the Azure SQL Database service ##  

In this task, you will enable a client application to access the Azure SQL Database service. This will be done by setting up the required authentication and acquiring the Application ID and Secret needed to authenticate the application. 

In the Azure portal, in the Search resources, services, and docs text box at the top of the Azure portal page, type App Registrations and press the Enter key. 
 
On the App Registrations blade, click + New registration. 

On the Register an application blade, specify the following settings (leave all others with their default values): 

Setting Value 
Name sqlApp  
Redirect URI (optional) Web and https://sqlapp  
On the Register, an application blade, click Register. 

Note: Once the registration is completed, the browser will automatically redirect you to sqlApp blade. 

On the sqlApp blade, identify the value of the Application (client) ID. 

On the sqlApp blade, in the Manage section, click Certificates & secrets. 

On the **sqlApp Certificates & secrets** blade / Client Secrets section, click + New client secret 

In the Add a client secret pane, specify the following values: 
Description - Key1 
Expires - 12 months 
Click Add to update the application credentials. 

On the **sqlApp Certificates & secrets** blade, make a note of the value of Key1. 

<br>

## 3.2 Create a policy allowing the application access to the Key Vault ##  
The aim here is to grant the newly registered app permissions to access secrets stored in the Key Vault.  

* ## Create a variable to store App ID 
In the powershell console, run the command
*<ins>$applicationId = 'a521ea6f-08b6-441f-9de4-825c71cf95ca'<ins>* 

<br>

* ## Create a variable to store KeyVault name 
*<ins>$kvName = (Get-AzKeyVault -ResourceGroupName 'rg-SecOps').VaultName 
$kvName<ins>* 

<br>

* ## grant permissions on the Key Vault to the application you registered 
Run script to grant relevant permission  
*<ins>Set-AZKeyVaultAccessPolicy -VaultName $kvName -ResourceGroupName rg-SecOps -ServicePrincipalName $applicationId -PermissionsToKeys get,wrapKey,unwrapKey,sign,verify,list<ins>*  
<br>

## 3.3 Retrieve SQL Azure database ADO.NET Connection String ##
Make a note of the ADO.NET (SQL authentication) connection string.  
This can be found on the SQL database overview blade. 

<br>

## 3.4 Log on to the Azure VM running Visual Studio 2019 and SQL Management Studio 19 ## 
![](/imgs/09RDP.png)  

<br>

## 3.5 Create a table in the SQL Database and select data columns for encryption ##
* Add a Firewall rule 
![](/imgs/10FWrule.png)
<br>

* Connect to the SQL Database with SQL Server Management Studio (SSMS) from the VM and create a table.
![](/imgs/11SSMS.png)
<br>

Thereafter, two data columns (SSN and Birthdate) will be encrypted using an autogenerated key from the Azure Key Vault.  
![](/imgs/12GetImage12.png)  

* ### Create Patients table 
Import SQL table into the database.  
![See 03 Patients.sql](https://github.com/Ladcze/implement-secure-data_always-encrypted/blob/3305b02e4c12d29a2178225ab6bdea30529823bd/03%20Patients.sql)  
This will create a Patients table. 

Note: The Always Encrypted Keys subnode contains the Column Master Keys and Column Encryption Keys subfolders.  
![](/imgs/16GetImage16.png)  

*** 
***

## Demonstrate the use of Azure Key Vault in encrypting the Azure SQL database.   

* ## Run a data-driven application to demonstrate the use of Azure Key Vault in encrypting the Azure SQL database ##
You will create a Console application using Visual Studio to load data into the encrypted columns and then access that data securely using a connection string that accesses the key in the Key Vault. 

* RDP onto the the az5-10-vm1 and launch Visual Studio 2019  
* Click sign in and authenticate using relevant credentials linked to the Azure subscription.  
* On the Get Started page, click Create a new project. 
* In the list of project templates, search for Console App (.NET Framework), in the list of results, click Console App (.NET Framework) for C#, and click Next. 
* On the Configure your new project page, specify the following values as indicated below  (leave other settings with their default values), then click Create:  
Project name - OpsEncrypt  
Solution name - OpsEncrypt  
Framework - .NET Framework 4.7.2  

* In the Visual Studio console, click the Tools menu, in the drop-down menu, click NuGet Package Manager, and, in the cascading menu, click Package Manager Console. 
 
* Install required NuGet components:
In the Package Manager Console pane, run the following code to install the first required NuGet package:   
*<ins>Install-Package Microsoft.SqlServer.Management.AlwaysEncrypted.AzureKeyVaultProvider<ins>*

* In the Package Manager Console pane, run the following to install the second required NuGet package: 
*<ins>Install-Package Microsoft.IdentityModel.Clients.ActiveDirectory<ins>*  

* Navigate to code and copy to the clipboard    
![See 04 program.cs](https://github.com/Ladcze/implement-secure-data_always-encrypted/blob/3305b02e4c12d29a2178225ab6bdea30529823bd/04%20program.cs)
 
* Return to the RDP session, and in the Visual Studio console, in the Solution Explorer window, click Program.cs and replace its content with the code you copied above.

* In the Visual Studio console, click the Start button to initiate the build of the console application and start it. 

The application will start a Command Prompt window. When prompted for a password, type the password that you specified in the deployment in Exercise 1 to connect to the Azure SQL Database.  
![](/imgs/17GetImage17.png)  

* Leave the console app running and switch to the SQL Management Studio console. 

* In the Object Explorer pane, right-click the medical database and, in the right-click menu, click New Query. 

From the query window, run the following query to verify that the data loaded into the database from the console app is encrypted.  
Sql statement:  
*<ins>SELECT FirstName, LastName, SSN, BirthDate FROM Patients;<ins>*

Switch back to the console application where you are prompted to enter a valid SSN. This will query the encrypted column for the data. At the Command Prompt, type the following and press the Enter key: 
![](/imgs/19GetImage19.png)
 

Command:  
*<ins>999-99-0003<ins>*

Note: Verify that the data returned by the query is not encrypted. 
![](/imgs/20GetImage20.png) 
 

To terminate the console app, press the Enter key  

<br> 

---

## Summary   
This proof of concept primarily focuses on the security features provided by Azure by implementing an always-encrypted database which is hosted on Azure's SQL offerings. 

---

## Reference
http://www.microsoft.com/learning
