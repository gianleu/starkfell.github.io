---
layout: post
comments: true
title: "In Progress... Setting up Continuous Deployment to Nano Server in Azure - Part 3"
date: 2016-12-05
---

This blog post will cover how to create a new Webhook for an Azure Runbook, store the Webhook in Azure Key Vault and then add the Webhook to
GitHub without the Webhook ever appearing in clear text during this process. 


# Overview

This article is the third in a series of blog posts on setting up continuous deployment to Nano Server in Azure. 

* [Setting up Continuous Deployment to Nano Server in Azure - Part 1](http://starkfell.github.io/continuous-deployment-to-nano-server-in-azure-p1/)
* [Setting up Continuous Deployment to Nano Server in Azure - Part 2](http://starkfell.github.io/continuous-deployment-to-nano-server-in-azure-p2/)
* [Setting up Continuous Deployment to Nano Server in Azure - Part 3](http://starkfell.github.io/continuous-deployment-to-nano-server-in-azure-p3/)

This article will cover the following:

* Create a new Webhook for the existing Azure Runbook (rb-Deploy-CoreWebAppDemo-To-Nano-Server) using PowerShell
* Store the Webhook URL in an existing Azure Key Vault
* Create a new Personal Access Token in GitHub
* Add the Webhook to GitHub using the GitHub API and PowerShell

# Prerequisites

**All of the resources that were deployed in Azure in the previous parts of this series are required to be in place before continuing.**

Additionally the following items are required.

* Follow the instructions below on a host running Windows 8.1 and higher or Windows Server 2016 RTM.
* Verify that you have access to a Co-Administrator or an Azure Organizational Account with access to an existing Azure Subscription.
* A basic understanding of GitHub as well as access to a Free GitHub Account is required.

# Create a secure Webhook for the  Runbook

Open up an elevated PowerShell prompt and login to Azure.

```powershell
Add-AzureRmAccount
```

Make sure to change over to the Subscription where you deployed the resources in the previous posts.

```powershell
Select-AzureRmSubscription -SubscriptionId <SUBSCRIPTION_ID>
```

Create a new Webhook for the Azure Runbook, **rb-Deploy-CoreWebAppDemo-To-Nano-Server**.

Syntax:

```powershell
$WebhookURI = (New-AzureRmAutomationWebhook `
    -Name secure-webhook `
    -RunbookName rb-Deploy-CoreWebAppDemo-To-Nano-Server `
    -IsEnabled:$true `
    -ExpiryTime (Get-Date).AddYears(1) `
    -ResourceGroupName <AZURE_AUTOMATION_ACCOUNT_RESOURCE_GROUP> `
    -AutomationAccountName <AZURE_AUTOMATION_ACCOUNT> `
    -Force).WebhookURI
```

Example:

```powershell
$WebhookURI = (New-AzureRmAutomationWebhook `
    -Name secure-webhook `
    -RunbookName rb-Deploy-CoreWebAppDemo-To-Nano-Server `
    -IsEnabled:$true `
    -ExpiryTime (Get-Date).AddYears(1) `
    -ResourceGroupName nano-automation `
    -AutomationAccountName nano-automation `
    -Force).WebhookURI
```

The Webhook URI that is normally shown when you run this command is stored in the **$WebhookURI** variable. At this point, you can echo out the variable
in any manner you choose if you want to see the Webhook URL.

# Add the new Webhook to Azure Key Vault

In order to store the Webhook URI in Azure Key Vault, the value of the Webhook needs to be converted into a secure string.

```powershell
$SecuredWebhookURI = ConvertTo-SecureString -String $WebhookURI -AsPlainText -Force
```

Add the Webhook URI to the Azure Key Vault created in **[Part 1](http://starkfell.github.io/continuous-deployment-to-nano-server-in-azure-p1/)**

Syntax:

```powershell
Set-AzureKeyVaultSecret `
    -VaultName <KEY_VAULT_NAME> `
    -Name webhook-uri `
    -SecretValue $SecuredWebhookURI
```

Example:

```powershell
Set-AzureKeyVaultSecret `
    -VaultName nanokeyvaultf4ac `
    -Name webhook-uri `
    -SecretValue $SecuredWebhookURI
```

# Create a new Personal Access Token in GitHub

In order to add the Webhook URL programatically to GitHub, a Personal Access Token needs to be generated as an alternative for using Basic Authentication. Additionally,
we can scope the type of access the Personal Access Token has in the GitHub Account as well as remove it at any point in time in the future.

Start by going to **[https://github.com/settings/tokens](https://github.com/settings/tokens)** and logging into your GitHub Account.

You should be on the **Personal Access tokens** page, click on the **Generate new token** button.

![continuous-deployment-to-nano-server-in-azure-p3-001]({{ site.github.url }}/media/continuous-deployment-to-nano-server-in-azure-p3-001.jpg)

In the **Token description** section type in **azure automation webhook**. Grant the webhook **admin:repo_hook** rights and then scroll down to the bottom of the page
and click on the **Generate token** button.

![continuous-deployment-to-nano-server-in-azure-p3-002]({{ site.github.url }}/media/continuous-deployment-to-nano-server-in-azure-p3-002.jpg)

# Add the new Personal Access Token to Azure Key Vault

Copy the Personal Access Token and convert it into a secure string in the elevated PowerShell prompt from earlier.

```powershell
$SecuredWebhookURI = ConvertTo-SecureString -String "fff9af39fe47f89e54e87866adc1ee9e9dec000d" -AsPlainText -Force
```

Syntax:

```powershell
Set-AzureKeyVaultSecret `
    -VaultName <KEY_VAULT_NAME> `
    -Name github-pat `
    -SecretValue $SecuredWebhookURI
```

Example:

```powershell
Set-AzureKeyVaultSecret `
    -VaultName nanokeyvaultf4ac `
    -Name github-pat `
    -SecretValue $SecuredWebhookURI
```

# Add the Webhook to GitHub from Azure Key Vault using PowerShell

In the same elevated PowerShell prompt from earlier, retrieve the value of the Personal Access Token in clear text and leave it in a variable.

```powershell
$Token = (Get-AzureKeyVaultSecret -VaultName nanokeyvaultf4ac -Name github-pat).SecretValueText
```

Retrieve the value of the Webhook URI from the Azure Key Vault in clear text but leave it in a variable.

```powershell
$WebhookURI = (Get-AzureKeyVaultSecret -VaultName nanokeyvaultf4ac -Name webhook-uri).SecretValueText
```

Create the payload to add the webhook to GitHub.

```powershell
$GitHub_Webhook = @{
    name = "web"
    active = $true
    events=@("push")
    config=@{
        content_type = "json"
        insecure_ssl = 0
        url = "$WebhookURI"
    }
}
```

Convert the payload to JSON.

```powershell
$GitHub_Webhook_JSON = $GitHub_Webhook | ConvertTo-Json
```

Add the Webhook to GitHub using the GitHub API from PowerShell.

Syntax:

```powershell
Invoke-RestMethod `
    -Method Post `
    -Uri https://api.github.com/repos/<GITHUB_USERNAME>/<REPOSITORY_NAME>/hooks `
    -Body $GitHub_Webhook_JSON `
    -Headers @{ "Authorization" = "token $Token" } | Out-Null
```

Example:

```powershell
Invoke-RestMethod `
    -Method Post `
    -Uri https://api.github.com/repos/starkfell/nano-deploy-demo/hooks `
    -Body $GitHub_Webhook_JSON `
    -Headers @{ "Authorization" = "token $Token" } | Out-Null
```

Go to the repository settings of where you added the Webhook and verify it is there.


Finally, you can run the command below to see the details of the Webhook in GitHub.

Syntax:

```powershell
Invoke-RestMethod `
    -Method Get `
    -UseBasicParsing `
    -Uri https://api.github.com/repos/<GITHUB_USERNAME>/<REPOSITORY_NAME>/hooks `
    -Headers @{ "Authorization" = "token $Token" }
```

Example:

```powershell
Invoke-RestMethod `
    -Method Get `
    -UseBasicParsing `
    -Uri https://api.github.com/repos/starkfell/nano-deploy-demo/hooks `
    -Headers @{ "Authorization" = "token $Token" }
```

## Closing

In this article we covered how to deploy an Azure Runbook to the Azure Automation account created in the previous article
and how to configure it to trigger the deployment of a .NET Application from Github to the Nano Server using a Webhook.

The next article will cover the following:

* Parsing Webhook Data from GitHub in the Azure Automation Runbook
* Securing Webhooks in Azure Automation