---
layout: post
title: "Bicep nested for loops"
date: 2023-06-13
categories: BICEP AZURE IAC
---

# Nested for loops in Bicep

Nested for loops are not natively supported in Bicep.
However, it is possible to get around this by using nested modules. Caution: do not nest too much though since this does decrease readability and increase complexity.

Lets say we have a JSON that looks something like this, this is application groups for Azure Virtual Desktop.

```
{
    "applicationsGroups": [
        {
            "name": "ApplicationGroupAdmin",
            "apps": [
                {
                    "name": "Remote Desktop Connection",
                    "properties": {
                        "applicationType": "InBuilt",
                        "commandLineArguments": "",
                        "commandLineSetting": "Allow",
                        "filepath": "C:\\windows\\system32\\mstsc.exe",
                        "iconpath": ""
                    }
                },
                {
                    "name": "Active Directory Users and Computers",
                    "properties": {
                        "applicationType": "InBuilt",
                        "commandLineArguments": "",
                        "commandLineSetting": "Allow",
                        "filepath": "C:\\Windows\\System32\\dsa.msc",
                        "iconpath": ""
                    }
                }
            ]
        },
        {
            "name": "ApplicationGroupBrowser",
            "apps": [
                {
                    "name": "Edge",
                    "properties": {
                        "applicationType": "InBuilt",
                        "commandLineArguments": "",
                        "commandLineSetting": "Allow",
                        "filepath": "C:\\Program Files (x86)\\Microsoft\\Edge\\Application\\msedge.exe",
                        "iconpath": ""
                    }
                }
            ]
        }
    ]
}
```

We create the application groups in another module. Here we loop through the application groups and provide the applications to the nested module.

```
var applicationGroups = loadJsonContent('../applicationGroups.json')

module application 'modules/application.bicep' = [for ag in applicationGroups: {
  name: toUpper('AVD-${ag.name}-APPLICATIONS')
  params: {
    applications: ag.apps
    applicationGroup: ag.name
  }
  dependsOn: [
    applicationGroup
  ]
}]
```

The nested module loops through the application array to create the applications and associate them to the application group.

```
param applications array
param applicationGroup string
param showInPortal bool = true

module application '../modules/avd/application.bicep' = [for app in applications: {
  name: toUpper('AVD-${applicationGroup}-${replace(app.name, ' ', '-')}-APP')
  params: {
    applicationGroupName: applicationGroup
    applicationType: app.properties.applicationType
    commandLineArguments: app.properties.commandLineArguments
    commandLineSetting: app.properties.commandLineSetting
    filepath: app.properties.filepath
    iconPath: empty(app.properties.iconpath) ? app.properties.filepath : app.properties.iconpath
    name: app.name
    showInPortal: showInPortal
  }
}]
```