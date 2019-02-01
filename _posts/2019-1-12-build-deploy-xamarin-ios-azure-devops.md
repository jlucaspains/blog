---
layout: post
title: "Build and Deploy Xamarin.iOS to Test Flight with Azure DevOps"
date: 2018-7-29
comments: true
sharing: true
categories: [xamarin-iOS]
---

The goal here is to build an Xamarin.iOS app from a git repository, create a releasable .ipa file and push it to the App Store Connect for Test Flight.

## Requirements
* A paid [apple developer account](https://developer.apple.com/)
* Working Xamarin.iOS project in a Source Control system that Azure DevOps can access
   * If you need a sample application, you can grab one [here](https://github.com/jlucaspains/BlogSamples/tree/master/Xamarin/LPains.LazyLoadedMasterDetailPage)
* Azure DevOps account and project (it is free, go get one)

## Register app ID and get certificate and provisioning profile
At the end of this section you should have .p12 signing certificate and a .mobileprovision provisioning profile. There are some good content about these next steps so I will just refer to them:

* [Create an app id](https://customersupport.doubledutch.me/hc/en-us/articles/229488228-iOS-How-to-Create-an-App-ID)
* [Create a distribution certificate](https://support.staffbase.com/hc/en-us/articles/115003458931-Creating-the-iOS-Distribution-Certificate)
* [Create a provisioning profile](https://clearbridgemobile.com/how-to-create-a-distribution-provisioning-profile-for-ios/)

## Create a build
The build pipeline is the heart of this post. I'm assuming you are familiar with Azure DevOps pipelines and such. Follow below steps to get the build setup:

1. Create a new build pipeline using the visual designer (it is easier to import the certificate and provisioning profile this way)
2. These are all the steps you will need in your pipeline

![Build Steps]({{ site.url }}/images/posts/iOSAzureDevOpsBuildAllSteps.png)

3. Install the distribution certificate
   1. Import your .p12 file in the Certificate field
   2. Create a pipeline variable named P12Password with the .p12 password and set it as secret

![Install Distribution Certificate]({{ site.url }}/images/posts/iOSAzureDevOpsBuildInstallCert.png)

4. Install the provisioning profile
   1. Import your .mobileprovisioning

![Install Provisioning Profile]({{ site.url }}/images/posts/iOSAzureDevOpsBuildInstallProfile.png)

5. Modify the info.plist file (optional)
   1. I use this to change app bundle id, icon sets for QA and PRD environments and finally set the app build number.
   2. To use the BuildNumber in the app version, you will need to set the Build Number Format to `$(date:yyyy.MM)$(rev:.r)`. This exp
   3. You may add or remove steps in the bash script below (provided for easy copy/paste)

![Modify info.plist]({{ site.url }}/images/posts/iOSAzureDevOpsBuildInfoPlist.png)

<pre class="brush: bash">
#!/bin/bash
echo "setting plist"

echo "setting CFBundleVersion to $(Build.BuildNumber)"
echo "setting CFBundleDisplayName to AppName"
echo "setting CFBundleIdentifier to com.Company.AppName" 

export INFO="$BUILD_SOURCESDIRECTORY/Path/To/Info.plist"

/usr/libexec/PlistBuddy -c "set :CFBundleVersion $(Build.BuildNumber)" $INFO
/usr/libexec/PlistBuddy -c "set :CFBundleDisplayName AppName" $INFO
/usr/libexec/PlistBuddy -c "set :CFBundleIdentifier com.Company.AppName" $INFO
/usr/libexec/PlistBuddy -c "set :XSAppIconAssets AppIcons_PRD.xcassets/AppIcons_PRD.appiconset" $INFO
/usr/libexec/PlistBuddy -c "set :XSLaunchImageAssets AppIcons_PRD.xcassets/LaunchImages_PRD.launchimage" $INFO
/usr/libexec/PlistBuddy -x -c "Print" $INFO
</pre>


6. Install the correct version of nuget
   1. Pick the version that works when building locally

![Nuget Install]({{ site.url }}/images/posts/iOSAzureDevOpsBuildNugetInstall.png)

7. Restore packages

![Nuget restore]({{ site.url }}/images/posts/iOSAzureDevOpsBuildNugetRestore.png)

8. Build!
   1. Make sure to select the right build configuration.
   2. Check the Create app package option
   3. Provide `$(APPLE_CERTIFICATE_SIGNING_IDENTITY)` in Signing Identity. This is an automatic variable output from the Install Certificate step
   4. Provide `$(APPLE_PROV_PROFILE_UUID)` in the Provisioning Profile UUID. This is an automatic variable output from the Install Provisioning Profile step

![Build Project]({{ site.url }}/images/posts/iOSAzureDevOpsBuildNugetRestore.png)

9. Copy the .ipa to staging

![Copy Fiels to Staging]({{ site.url }}/images/posts/iOSAzureDevOpsBuildCopyFiles.png)

10. Publish Artifacts 

![Publish staging]({{ site.url }}/images/posts/iOSAzureDevOpsBuildPublishArtifacts.png)

## Create a release
The release is much simpler. All you need is to push the `.ipa` to Test Flight. There is a couple of things to be aware first though:

* You need to [create the app](https://help.apple.com/app-store-connect/#/dev2cd126805) in App Connect Store first.
* You need to manually push the app via [Application Loader](https://help.apple.com/itc/apploader/) at least once before the release will work.
* The release uses fastlane so if you use a private MacOS for builds you may need to install it. Check the fastline [guide](https://docs.fastlane.tools/getting-started/ios/setup/).

The release will look like something like this:
![Release]({{ site.url }}/images/posts/iOSAzureDevOpsRelease.png)

## Special considerations
1. Note that both nuget restore and xamarin build were executed over the .csproj file instead of the solution. I'm not sure why, but everytime I use the .sln the build fails.
2. Apple store require you to provide export compliance documentation and will not automatically publish the app for testing without this. There are ways around this. [Read about it](https://help.apple.com/app-store-connect/#/dev88f5c7bf9).