---
layout: post
title: "Integration Runtime Windows Authentication"
date: 2023-10-10
categories: AZURE WINDOWS_SERVER WINDOWS INTEGRATION_RUNTIME DATA_FACTORY
---

# Using Windows Authentication with azure data factory integration runtime fails

Encountered this issue when having an integration runtime running on an azure VM trying to access an SQL server also running on an azure VM using Windows Authentication.

Got the following error:

```
Cannot connect to SQL Database. Please contact SQL server team for further support. Server: '<IP>', Database: '<DBNAME>', User: '<domain\username>'. Check the linked service configuration is correct, and make sure the SQL Database firewall allows the integration runtime to access.
Login failed. The login is from an untrusted domain and cannot be used with Integrated authentication., SqlErrorNumber=18452,Class=14,State=1,
```

Both virtual machines are in the same Active Directory domain. The error message feels like a red herring since they both are in the same domain, there shouldn't be any trust issues. I could login locally on both vms with the service account to access the database.

The solution for me was the following:

GPO: Policies -> Windows Settings -> Security Settings -> User Rights Management -> Allow log on locally

Add the service account/user to the list and apply the policy to the SQL server in this case. After that it started working.
