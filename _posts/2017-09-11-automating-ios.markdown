---
layout: post
title:  "Automating iOS Tasks from a Mac's Terminal"
subtitle: "Creating Command Line Versions of Xcode Functionalities"
date:   2017-09-11
author: Lior Halphon
---

One of the common tasks an iOS developer would want to perform is launching an application on a device from a connected Mac. Usually, this is done from Xcode, which can build, install and launch an application. However, this only covers the very basic scenario – launching an app you just built on a device. But what if you want to launch an application via command line, or a script? What if you're not even using Xcode for your project? This functionality is very critical, for example, for automatic testing. It enables automatically testing hundreds of applications in nightly cycles that guarantee top quality reliability, and you can't test an app before launching it first.

### When there's no official tool
Sadly, in these cases Apple provides no official solution. The lack of an official solution led to the creation of several third party tools that can launch iOS applications via command line, however none of them were reliable or versatile enough for our needs, so I ended up creating my own tool, [iLaunch](https://github.com/Appdome/iLaunch). During iLaunch's development, more capabilities not directly related to launching applications were added. As of today iLaunch is capable of:

 * Launching both user-installed and system applications
   * Providing environment variables, which can be used, for example, to debug [dyld](https://developer.apple.com/legacy/library/documentation/Darwin/Reference/ManPages/man1/dyld.1.html) or [Objective-C use-after-free bugs](https://developer.apple.com/library/content/technotes/tn2239/_index.html#//apple_ref/doc/uid/DTS40010638-CH1-SUBSECTION30)
   * Providing command-line arguments to the application
   * Optionally forwarding the application's stdout and stderr (And effectively NSLog calls) to the local terminal. This is extremely useful as it does not include logs from other processes, especially on iOS 10 and newer where the complete log is constantly being spammed by system processes.
 * Listing all user-installed and system applications, including their bundle identifier, display name, and installation path.
 * Listing information about all connected devices
 * Extracting an application's sandbox (if its provisioning profile allows doing so)
 * Extracting, syncing and deleting crash logs (much faster and more convenient than using Xcode and iTunes for the task)
 * Saving a device screenshot to a PNG file

### iLaunch: Behind the Scenes
The way iLaunch works is not by reimplementing the USB-based protocol used by Xcode and iOS to communicate – this is too much work, and protocol changes make maintenance difficult. Instead, I link against Xcode's own libraries and use its own code to achieve our goals. However, this is often not as simple as it looks.

The first challenge was actually linking against Xcode's libraries. Xcode's libraries heavily use [@rpath](https://developer.apple.com/library/content/documentation/DeveloperTools/Conceptual/DynamicLibraries/100-Articles/RunpathDependentLibraries.html) which makes it kind of tricky to link against without hardcoding a specific path to Xcode. Since rpaths can not be modified in runtime, I eventually used a dirty trick – setting our rpath to a temporary symlink to Xcode.

Another kind of challenge is that some methods can only be used after some kind of initialization occurs, or must run in a specific thread, or require the main NSRunLoop to run. Take the following code for example:

    NSSet *devices = [DVTiOSDevice alliOSDevices];

If the main NSRunLoop never runs, it will always return an empty set, because the code that updates the (cached) return value of this method runs in the main NSRunLoop. Since iLaunch is not run loop based, it must be initiated manually for a few seconds every now and then so the return values of such methods are actually meaningful.

In addition to some slightly Xcode-specific problems, there are the usual challenges that pop up when it comes to using private Apple APIs. Figuring out what the correct values for method arguments is not always easy, and Apple's tendency to write methods with dozens of arguments in their private code doesn't make it easier. Take this 27-argument long selector for example:

    +[IDELaunchParametersSnapshot launchParametersWithSchemeIdentifier:launcherIdentifier:debuggerIdentifier:launchStyle:runnableLocation:debugProcessAsUID:workingDirectory:commandLineArgs:environmentVariables:architecture:platformIdentifier:buildConfiguration:buildableProduct:deviceAppDataPackage:allowLocationSimulation:locationScenarioReference:showNonLocalizedStrings:language:region:routingCoverageFileReference:enableGPUFrameCaptureMode:enableGPUValidationMode:debugXPCServices debugAppExtensions:internalIOSLaunchStyle:internalIOSSubstitutionApp:launchAutomaticallySubstyle:]

There's also the opposite problem – sometimes methods are not customizable _enough_. For example, the method that takes a screenshot will always call a method that adds it to some sort of screenshot library, which will also save it a path it generates instead of a path I provide. Another problem is that the function that extracts crash logs will copy the entire 800MB `dyld_shared_cache` over and over again each time it is called, although it's not even used. Problems of this kind are solved by us hooking methods called by the method _I_ call so I can alter unwanted behavior.

### Beyond iLaunch, Beyond Xcode

These techniques can be used to create command line versions of pretty much any Xcode functionality. They're not even limited to Xcode – they can be used with pretty much any Cocoa application, making them extremely useful when there's no suitable command line tool that can perform a certain task. The possibilities are even greater since not only does macOS allow you to dynamically link against dylib files and frameworks, it also allows you to dlopen executables! And with the dynamic nature of Objective C, the possibilities are endless.

iLaunch can be compiled and used on any recent macOS and Xcode version, and is available under the MIT license on [GitHub](https://github.com/Appdome/iLaunch).