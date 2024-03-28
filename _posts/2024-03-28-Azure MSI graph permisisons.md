---
layout: post
title: "Give azure user-assigned managed identity access to graph"
date: 2024-03-28
categories: AZURE MSI IDENTITY GRAPH MANAGED_IDENTITY
---

# Give azure user-assigned managed identity access to graph

If you require a user-assigned managed identity to access graph it cannot to my knowledge be done via the portal.
You need to find the service principal id using graph.

Here I'm using the graph module in powershell to get the id from the msi

```
$displayname = "msi-test"
$spId = Get-MgServicePrincipal -ConsistencyLevel eventual -Search "DisplayName:$displayName" | select-object id
```

I also need to get the approle ids that are available.
There is a Microsoft Graph Application in your tenant that always have the same guid.

```
$mgGraphAppId = "00000003-0000-0000-c000-000000000000"
$roleData = Get-MgServicePrincipal -ConsistencyLevel eventual -Search "appId:$mgGraphAppId" | Select-Object appDisplayName, appId, appRoles
$roleId = $roleData.appRoles | Where-Object { $_.value -eq $roleValue } | Select-Object -ExpandProperty Id
```

Now we need to get the app id

```
$mgGraphId = Get-MgServicePrincipal -ConsistencyLevel eventual -Search "appId:$mgGraphAppId" | Select-Object -ExpandProperty Id 
```

Putting it all together and adding the role

```
$roleId = Application.Read.All
$params = @{
    principalId = $spId
    resourceId  = $mgGraphId
    appRoleId   = $roleId
}
New-MgServicePrincipalAppRoleAssignment -ServicePrincipalId $spId -BodyParameter $params
```
