---
title: Configure WAF rate limit rule for Front Door - Azure PowerShell
description: Learn how to configure a rate limit rule for an existing Front Door endpoint.
author: vhorne
ms.service: web-application-firewall
ms.topic: article
services: web-application-firewall
ms.date: 03/21/2022
ms.author: victorh 
ms.custom: devx-track-azurepowershell
---

# Configure a Web Application Firewall rate limit rule using Azure PowerShell

The Azure Web Application Firewall (WAF) rate limit rule for Azure Front Door controls the number of requests allowed from a particular client IP address to the application during a rate limit duration. This article shows how to configure a WAF rate limit rule that controls the number of requests allowed from a particular client to a web application that contains */promo* in the URL using Azure PowerShell.


If you don't have an Azure subscription, create a [free account](https://azure.microsoft.com/free/?WT.mc_id=A261C142F) before you begin.

> [!NOTE]
> Rate limits are applied for each client IP address. If you have multiple clients accessing your Front Door from different IP addresses, they will have their own rate limits applied.

## Prerequisites

Before you begin to set up a rate limit policy, set up your PowerShell environment and create a Front Door profile.

### Set up your PowerShell environment

Azure PowerShell provides a set of cmdlets that use the [Azure Resource Manager](../../azure-resource-manager/management/overview.md) model for managing your Azure resources. 

You can install [Azure PowerShell](/powershell/azure/) on your local machine and use it in any PowerShell session. Follow the instructions on the page, to sign in with your Azure credentials, and install Az PowerShell module.

#### Connect to Azure with an interactive dialog for sign in

```
Connect-AzAccount

```
Before install Front Door module, make sure the current version of PowerShellGet is installed. Run the following command and reopen PowerShell.

```
Install-Module PowerShellGet -Force -AllowClobber
``` 

#### Install Az.FrontDoor module 

```
Install-Module -Name Az.FrontDoor
```
### Create a Front Door profile

Create a Front Door profile by following the instructions described in [Quickstart: Create a Front Door profile](../../frontdoor/quickstart-create-front-door.md)

## Define URL match conditions

Define a URL match condition (URL contains /promo) using [New-AzFrontDoorWafMatchConditionObject](/powershell/module/az.frontdoor/new-azfrontdoorwafmatchconditionobject).
The following example matches */promo* as the value of the *RequestUri* variable:

```powershell-interactive
   $promoMatchCondition = New-AzFrontDoorWafMatchConditionObject `
     -MatchVariable RequestUri `
     -OperatorProperty Contains `
     -MatchValue "/promo"
```
## Create a custom rate limit rule

Set a rate limit using [New-AzFrontDoorWafCustomRuleObject](/powershell/module/az.frontdoor/new-azfrontdoorwafcustomruleobject). 
In the following example, the limit is set to 1000. Requests from a particular client IP address client to the promo page exceeding 1000 during one minute are blocked until the next minute starts.

```powershell-interactive
   $promoRateLimitRule = New-AzFrontDoorWafCustomRuleObject `
     -Name "rateLimitRule" `
     -RuleType RateLimitRule `
     -MatchCondition $promoMatchCondition `
     -RateLimitThreshold 1000 `
     -Action Block -Priority 1
```

## Configure a security policy

Find the name of the resource group that contains the Front Door profile using `Get-AzureRmResourceGroup`. Next, configure a security policy with a custom rate limit rule using [New-AzFrontDoorWafPolicy](/powershell/module/az.frontdoor/new-azfrontdoorwafpolicy) in the specified resource group that contains the Front Door profile.

The below example uses the Resource Group name *myResourceGroupFD1* with the assumption that you've created the Front Door profile using instructions provided in the [Quickstart: Create a Front Door](../../frontdoor/quickstart-create-front-door.md) article, using [New-AzFrontDoorWafPolicy](/powershell/module/az.frontdoor/new-azfrontdoorwafpolicy).

```powershell-interactive
   $ratePolicy = New-AzFrontDoorWafPolicy `
     -Name "RateLimitPolicyExamplePS" `
     -resourceGroupName myResourceGroupFD1 `
     -Customrule $promoRateLimitRule `
     -Mode Prevention `
     -EnabledState Enabled
```
## Link policy to a Front Door front-end host

Link the security policy object to an existing Front Door front-end host and update Front Door properties. First retrieve the Front Door object using [Get-AzFrontDoor](/powershell/module/Az.FrontDoor/Get-AzFrontDoor) command.
Next, set the front-end *WebApplicationFirewallPolicyLink* property to the *resourceId* of the "$ratePolicy" created in the previous step using [Set-AzFrontDoor](/powershell/module/Az.FrontDoor/Set-AzFrontDoor) command. 

The below example uses the Resource Group name *myResourceGroupFD1* with the assumption that you've created the Front Door profile using instructions provided in the [Quickstart: Create a Front Door](../../frontdoor/quickstart-create-front-door.md) article. Also, in the below example, replace $frontDoorName with the name of your Front Door profile. 

```powershell-interactive
   $FrontDoorObjectExample = Get-AzFrontDoor `
     -ResourceGroupName myResourceGroupFD1 `
     -Name $frontDoorName
   $FrontDoorObjectExample[0].FrontendEndpoints[0].WebApplicationFirewallPolicyLink = $ratePolicy.Id
   Set-AzFrontDoor -InputObject $FrontDoorObjectExample[0]
 ```

> [!NOTE]
> You only need to set *WebApplicationFirewallPolicyLink* property once to link a security policy to a Front Door front-end. Subsequent policy updates are automatically applied to the front-end.

## Next steps

- Learn more about [Front Door](../../frontdoor/front-door-overview.md).
