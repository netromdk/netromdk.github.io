---
layout: post
title:  "Upgrading to Xcode 4.4 or later"
date:   2013-02-03 22:13:56 +0200
---

Xcode moved to the App Store as of version 4.4 and it had certain consequences to the common programmer. First of all, the SDKs are no longer located at "/Developer/SDKs" like they were before. The app is located at "/Applications/Xcode.app" on a default install and it contains the SDKs in the directory:

<p><blockquote>/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs</blockquote></p>

All command-line tools are no longer installed per default either so in order to install them open Xcode, go to "Preferences", select the "Downloads", choose "Components" and install "Command Line Tools".

In order to configure the BSD tools correctly it is needed to instruct where Xcode is installed. It is done like this:

```shell
xcode-select --switch /Applications/Xcode.app/Contents/Developer
```

Another important change of Xcode 4.4+ is the discontinuation of the 10.6 SDK. When installing, or updating, 4.4+ it will only have SDKs 10.7 and 10.8. If 10.6 is installed it will be <em>removed</em>. So if you have 4.3 or earlier installed I recommend that you backup 10.6 if you need it for development. The SDK contains a lot of symbolic links so it's important to <i>not</i> follow them:

```shell
zip -r9 --symlink /path/to/store/sdk-10.6.zip /Developer/SDKs/MacOSX10.6.sdk
```

If you don't have SDK 10.6 and want it you can follow these steps:
<ol>
	<li>Go to <a href="https://developer.apple.com/downloads/index.action" title="https://developer.apple.com/downloads/index.action" target="_blank">https://developer.apple.com/downloads/index.action</a></li>
	<li>You have to login which thus requires an Apple Developer account.</li>
	<li>Filter the search with "Xcode 4 and iOS SDK 4.3".</li>
	<li>Download "Xcode 4 and iOS SDK 4.3 - Final" DMG (4.28 GB).</li>
	<li>Open the DMG to mount it and go to "/Volumes/Xcode and iOS SDK/Packages".</li>
	<li>You will find the file "MacOSX10.6.pkg" there which contains the SDK.</li>
</ol>

I personally use SDK 10.6 on a daily basis for work so I have to have it installed at all times. And since Xcode will remove it if put into its SDK directory on update I have instead created the old directory "/Developer/SDKs" and put the SDK there. Then I put symbolic links to the new SDKs 10.7 and 10.8.
