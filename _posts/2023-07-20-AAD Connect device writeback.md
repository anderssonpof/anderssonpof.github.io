---
layout: post
title: "AAD Connect device writeback access denied"
date: 2023-07-20
categories: WINDOWS WINDOWS_SERVER AAD AAD_CONNECT AZURE
---

# Access denied when configuring device writeback

This one isn't that complicated when you think about it, but it would have been nice if Microsoft provided a little more insight or not forcing one to restart the process if it fails in this manner.

When you are configuring device writeback AAD connect used the account that started the config utility to authenticate to your Active Directory. This mean that in our case since we use LAPS and I started it as the local administrator it couldn't auth to AD.

Solution is to start azure ad connect with a AD user that has permissions to do everything required. Use the service account for Azure AD connect, create a temporary user with permissions to the RegisteredDevices container or if you are lazy use a da/ea account (not recommended).

