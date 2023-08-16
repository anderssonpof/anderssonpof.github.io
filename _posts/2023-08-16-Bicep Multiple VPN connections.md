---
layout: post
title: "Bicep Multiple VPN connections"
date: 2023-08-16
categories: BICEP AZURE IAC
---

# Deploying multiple VPN connections using Bicep fails

When deploying multiple VPN connections attached to one VPN site the deployment will timeout with an unhelpful error.

This is to my understanding caused by the asynchronous nature of the deployment. Azure resource manager tries to deploy both of the connections at the same time and therefore one of them will timeout.

To solve this deploy the vpn connection synchronously instead. Adding a depends on block that depends on the other connection solves this issue.

for example
```
module s2sConnection1 './modules/networking/vpnConnection.bicep' = {
  name: 'S2S-CONNECTION-01'
  params: {
    name: 'Connection-s2s-${weuLocation}-01'
    psk: kv.getSecret('s2s-${weuLocation}-psk')
    vpnSiteLinkName: s2sSite1.outputs.vpnSiteLinkName
    vpnSite: s2sSite1.outputs.name
    hubName: hubName
    vpngwName: s2s.outputs.name
  }
}

module s2sConnection2 './modules/networking/vpnConnection.bicep' = {
  name: 'S2S-CONNECTION-02'
  params: {
    name: 'Connection-s2s-${weuLocation}-02'
    psk: kv.getSecret('s2s-${weuLocation}-psk')
    vpnSiteLinkName: s2sSite2.outputs.vpnSiteLinkName
    vpnSite: s2sSite2.outputs.name
    hubName: hubName
    vpngwName: s2s.outputs.name
  }
  dependsOn: [
    s2sConnection1
  ]
}
```
