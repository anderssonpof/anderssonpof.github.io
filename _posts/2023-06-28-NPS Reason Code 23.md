---
layout: post
title: "NPS Reason Code 23"
date: 2023-06-28
categories: WINDOWS WINDOWS_SERVER NPS NETWORK_POLICY_SERVER
---

# NPS Reason Code 23

Reason Code 23 can mean quite a few different things. Taken from Microsoft documentation below:

An error occurred during the Network Policy Server use of the Extensible Authentication Protocol (EAP). Check EAP log files for EAP errors. By default, these log files are located at `%windir%\System32\Logfiles`

In our case the issue was related to auto-renewal of the server certificate. Apparently when the certificate is renewed the NPS server doesn't really use the new certificate.

To solve this issue you need to reapply the certificate to each network profile → constraints → authentication method that uses that certificate.
Choose a different certificate (create a self-signed if none exists) press OK and apply then go back and pick the correct certificate again.

I will just reiterate that this is one possible solutions to this error.
