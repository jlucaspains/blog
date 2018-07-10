---
layout: post
title: "Install Root CA cert in Android emulator"
date: 2018-7-7
comments: true
sharing: true
categories: xamarin-android
---

When developing a small Xamarin forms app I wanted the android emulator to connect to an intranet identity provider. That didn't work. The id server was using an SSL certificate issued by an untrusted CA. Here's what I did to get it working.

First, you will need the CA certificate so android can trust the SSL cert. If you don't have the cert, you can export it using chrome:

chrome on windows:
![Export Certificate]({{ site.url }}/images/posts/ExportCertificate.gif)

chrome on mac: https://stackoverflow.com/questions/25940396/how-to-export-certificate-from-chrome-on-a-mac

Next, push the .cer to the emulator using adb push command. Note that even though I'm pushing the cert to the sdcard it actually shows in a different location. 

Also, you need to enable device security in order to install the certificate. Just follow the prompts.

adb push path_to_cert\ca_cert_file.cer /sdcard/ca_cert_file.cer

Install the certificate using Settings app:
![Import Certificate]({{ site.url }}/images/posts/ImportCertificateAndroid.gif)

After import, you can check your certs in User credentials under Encryption & credentials.

