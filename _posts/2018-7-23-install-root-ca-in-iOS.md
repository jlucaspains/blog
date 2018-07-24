---
layout: post
title: "Install root CA in iOS 11+"
date: 2018-7-23
comments: true
sharing: true
category: [iOS]
---

This post is a follow up to [Install root CA in Android]({% post_url 2018-7-7-install-root-ca-in-android %}).

The problem is the same here and the solution is somewhat similar too. First, export the CA certificate like explained in the Android specific post.

Email the certificate as an attachment to an email account you can access via Safari on your device. Only Safari seems to allow the cert to install. Open the attachment and you will see the following prompt.

![Cert install confirmation]({{ site.url }}/images/posts/iOSSSLProfileInstallConfirmation.png)

Touch `Allow`. You may see a message about which device you want to install the certificate. If so, pick the device you need the cert installed.

![Chose a device]({{ site.url }}/images/posts/iOSSSLProfileDeviceChoice.png)

Next, an `Install Profile` page will display with the cert to be imported.

![Install profile]({{ site.url }}/images/posts/iOSSSLProfileInstall.png)

Touch `Install` and provide the device passcode. 

![Install profile warning]({{ site.url }}/images/posts/iOSSSLProfileInstallWarning.png)

Another warning will show about risks of installing an unmanaged certificate. If you trust the certificate, go ahead and touch `Install` again. You're almost done now.

Lastly, you will need to go to `Settings > General > About > Certificate Trust Settings` and toggle `enable full trust for root certificates`. Again, only do this if you trust the certificate origin.

![Install profile certificate trust settings]({{ site.url }}/images/posts/iOSSSLCertificateTrustSettings.png)

All done. You can now navigate SSL websites with certificates issued by the root CA you just installed.

Cheers!