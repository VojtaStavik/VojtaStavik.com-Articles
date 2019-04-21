---
layout: post
title: Building an iOS App Without Xcode's Build System
filename: "2018-10-15-building-ios-app-without-xcode.md"
---

A build system, despite its _scary-sounding_ name, is just a regular program, which knows how to build other programs. As an iOS developer, you're certainly familiar with how to build a project using Xcode. You go to the `Product` menu and select `Build`, or you use the `⌘B` keyboard shortcut.

You may have also heard about [Xcode Command Line Tools](https://developer.apple.com/library/archive/technotes/tn2339/_index.html). It's a set of tools which allows you to build Xcode projects directly from the terminal using the `xcodebuild` command. A very convenient thing for automating your processes, for example on your [CI](https://www.google.de/search?q=continuous+integration).

![Build System](/images/2018-10-15/build-system-1.png)

No matter how you've initiated it, the building itself is orchestrated by Xcode's build system.

**Can we replicate the building process and build the app "manually", without Xcode's build system?**

Is it possible to **sign** the resulting app? Or even **deploy** it to an actual iOS device?

<!-- more -->

> ⚠️ **Disclaimer 1** ⚠️
> #### What is this post about:
> Writing a non-reusable script that builds one concrete iOS project the simplest way possible.
> #### What is this post NOT about:
> Writing a complex and universal build system.

---

## Content
- [The App We're About to Build](#the-app-were-about-to-build)
- [**Step 1:** Prepare Working Folders](#step-1-prepare-working-folders)
- [**Step 2:** Compile Swift Files](#step-2-compile-swift-files)
- [**Step 3:** Compile Storyboards](#step-3-compile-storyboards)
- [**Step 4:** Process and Copy Info.plist](#step-4-process-and-copy-infoplist)
- [Running ExampleApp in the Simulator](#running-exampleapp-in-the-simulator)
- [Building for Device](#building-for-device)
- [**Step 5:** Copy Swift Runtime Libraries](#step-5-copy-swift-runtime-libraries)
- [**Step 6:** Code Signing](#step-6-code-signing)
- [Installing to an iOS Device](#installing-to-an-ios-device)
- [Next Steps](#next-steps)

---

### The App We're About to Build

I let Xcode 10.0 generate a new project using the `Single View App` template, and named it "ExampleApp". This is going to be the reference app we will try to build "manually". The only adjustment to the project I made was adding a `UILabel` with `🎉` to the main (and only) `ViewController`.

![Build System](/images/2018-10-15/build-system-4.png)

I also created the `build.bash` file in the root folder of the project. We're going use this file for the actual build script.

![Build System](/images/2018-10-15/build-system-2.png)

Don't forget to make the file executable by running the following in the terminal:
{% highlight sh %}
$ chmod +x build.bash
{% endhighlight %}

---

## Step 1: Prepare Working Folders

> ⚠️ **Disclaimer 2** ⚠️
>
> The complete "recipe" of how the app should be built is contained in its `xcodeproj` file. This article is not about how to parse and retrieve this information from it.
>
> We will ignore the project file for the sake of this article. To make our life easier, **we will hard-code all the details like the project name, source files, or build settings, directly into the build script.**

Let's start with some housekeeping. We need to define and create a set of folders we will be using during the building process.

{% highlight sh %}

#############################################################
#  build.bash
#############################################################

#!/bin/bash

# Exit this script immediately if any of the commands fails
set -e

PROJECT_NAME=ExampleApp

# The product of this script. This is the actual app bundle!
BUNDLE_DIR=${PROJECT_NAME}.app

# A place for temporary files needed for building
TEMP_DIR=_BuildTemp


#############################################################
echo → Step 1: Prepare Working Folders
#############################################################

# Delete existing folders from previous builds
rm -rf ${BUNDLE_DIR}
rm -rf ${TEMP_DIR}

mkdir ${BUNDLE_DIR}
echo ✅ Create ${BUNDLE_DIR} folder

mkdir ${TEMP_DIR}
echo ✅ Create ${TEMP_DIR} folder

{% endhighlight %}

When you run the script, your terminal should like this:
```
$ ./build.bash
→ Step 1: Prepare Working Folders
✅ Create ExampleApp.app folder
✅ Create _BuildTemp folder
```

Notice, that the script created two folders:

![Build System](/images/2018-10-15/build-system-3.png)

---

## Step 2: Compile Swift Files

Did you know you can call the Swift compiler directly from the terminal using `swiftc`?

{% highlight sh %}
$ swiftc -h
OVERVIEW: Swift compiler

USAGE: swiftc [options] <inputs>
...
{% endhighlight %}

With the correct flags, compiling all `.swift` source files is quite a straightforward operation.

{% highlight sh %}
#############################################################
echo → Step 2: Compile Swift Files
#############################################################

# The root directory of the project sources
SOURCE_DIR=ExampleApp

# All Swift files in the source directory
SWIFT_SOURCE_FILES=${SOURCE_DIR}/*.swift

# Target architecture we want to build for
TARGET=x86_64-apple-ios12.0-simulator

# Path to the SDK we want to use for compiling
SDK_PATH=$(xcrun --show-sdk-path --sdk iphonesimulator)

swiftc ${SWIFT_SOURCE_FILES} \
  -sdk ${SDK_PATH} \
  -target ${TARGET} \
  -emit-executable \
  -o ${BUNDLE_DIR}/${PROJECT_NAME}

echo ✅ Compile Swift source files ${SWIFT_SOURCE_FILES}
{% endhighlight %}

I'd like to talk a little bit about the `-emit-executable` flag. According to the documentation, when this flag is used, the compiler "emits a linked executable". When you run `swiftc` in the verbose mode (`-v`), you can actually see, what's happening under the hood. The compiler creates `.o` object for every `.swift` file, and then calls `ld` to link them into the final executable file.

Notice, that we set output (`-o`) to `ExampleApp.app/ExampleApp`. That's the resulting executable file from the compilation process.

Run the script and you should see the following:
```
$ ./build.bash
→ Step 1: Prepare Working Folders
✅ Create ExampleApp.app folder
✅ Create _BuildTemp folder

→ Step 2: Compile Swift Files
✅ Compile Swift source files ExampleApp/AppDelegate.swift ExampleApp/ViewController.swift
```

You can also check, that the resulting executable was created inside the `ExampleApp.app` bundle:
```
$ tree ExampleApp.app
ExampleApp.app
└── ExampleApp
```

---

## Step 3: Compile Storyboards

Source code files are not the only files which need to be compiled. We also need to compile all `.storyboard` files. The compiler for storyboards (and all the other interface builder files) is called `ibtool`. The resulting files from the process have the extension `.storyboardc`.

> **Fun fact**
>
> `storyboardc` files are file bundles. When you inspect the content of the bundle, you will see, that a compiled storyboard is actually just a bunch of `nib` files.

`ibtool` doesn't accept more than one file at a time. We need to use a for loop to iterate over all storyboards.

{% highlight sh %}
#############################################################
echo → Step 3: Compile Storyboards
#############################################################

# All storyboards in the Base.lproj directory
STORYBOARDS=${SOURCE_DIR}/Base.lproj/*.storyboard

# The output folder for compiled storyboards
STORYBOARD_OUT_DIR=${BUNDLE_DIR}/Base.lproj

mkdir -p ${STORYBOARD_OUT_DIR}
echo ✅ Create ${STORYBOARD_OUT_DIR} folder

for storyboard_path in ${STORYBOARDS}; do
  #
  ibtool $storyboard_path \
    --compilation-directory ${STORYBOARD_OUT_DIR}

  echo ✅ Compile $storyboard_path
done
{% endhighlight %}



Let's run the script and verify the content of `ExampleApp.app`:
```
$ ./build.bash
→ Step 1: Prepare Working Folders
✅ Create ExampleApp.app folder
✅ Create _BuildTemp folder

→ Step 2: Compile Swift Files
✅ Compile Swift source files ExampleApp/AppDelegate.swift ExampleApp/ViewController.swift

→ Step 3: Compile Storyboards
✅ Create ExampleApp.app/Base.lproj folder
✅ Compile ExampleApp/Base.lproj/LaunchScreen.storyboard
✅ Compile ExampleApp/Base.lproj/Main.storyboard
```

```sh
$ tree -L 2 ExampleApp.app
ExampleApp.app
├── Base.lproj
│   ├── LaunchScreen.storyboardc
│   └── Main.storyboardc
└── ExampleApp
```

---

## Step 4: Process and Copy Info.plist

A valid app bundle has to contain `Info.plist`. Unfortunately, we can't simply copy the one from the project directory. This _raw_ file contains several variables which we need to replace with the actual values.

```
Info.plist variables
----------------------------------------------
<key>CFBundleExecutable</key>
<string>$(EXECUTABLE_NAME)</string>
<key>CFBundleIdentifier</key>
<string>$(PRODUCT_BUNDLE_IDENTIFIER)</string>
<key>CFBundleName</key>
<string>$(PRODUCT_NAME)</string>
 ```

To deal easily with `plist` files, Apple provides us with a tool called [PlistBuddy](http://www.manpagez.com/man/8/PlistBuddy/). Because we don't want to modify the original `plist` we first create a temporary copy in the `_BuildTemp` folder, and modify it there. Finally, we copy the processed `plist` to the app bundle.


{% highlight sh %}
#############################################################
echo → Step 4: Process and Copy Info.plist
#############################################################

# The location of the original Info.plist file
ORIGINAL_INFO_PLIST=${SOURCE_DIR}/Info.plist

# The location of the temporary Info.plist copy for editing
TEMP_INFO_PLIST=${TEMP_DIR}/Info.plist

# The location of the processed Info.plist in the app bundle
PROCESSED_INFO_PLIST=${BUNDLE_DIR}/Info.plist

# The bundle identifier of the resulting app
APP_BUNDLE_IDENTIFIER=com.vojtastavik.${PROJECT_NAME}

cp ${ORIGINAL_INFO_PLIST} ${TEMP_INFO_PLIST}
echo ✅ Copy ${ORIGINAL_INFO_PLIST} to ${TEMP_INFO_PLIST}


# A command line tool for dealing with plists
PLIST_BUDDY=/usr/libexec/PlistBuddy

# Set the correct name of the executable file we created at step 2
${PLIST_BUDDY} -c "Set :CFBundleExecutable ${PROJECT_NAME}" ${TEMP_INFO_PLIST}
echo ✅ Set CFBundleExecutable to ${PROJECT_NAME}

# Set a valid bundle indentifier
${PLIST_BUDDY} -c "Set :CFBundleIdentifier ${APP_BUNDLE_IDENTIFIER}" ${TEMP_INFO_PLIST}
echo ✅ Set CFBundleIdentifier to ${APP_BUNDLE_IDENTIFIER}

# Set the proper bundle name
${PLIST_BUDDY} -c "Set :CFBundleName ${PROJECT_NAME}" ${TEMP_INFO_PLIST}
echo ✅ Set CFBundleName to ${PROJECT_NAME}

# Copy processed Info.plist to the app bundle
cp ${TEMP_INFO_PLIST} ${PROCESSED_INFO_PLIST}
echo ✅ Copy ${TEMP_INFO_PLIST} to ${PROCESSED_INFO_PLIST}
{% endhighlight %}

Let's run the script:

{% highlight sh %}
$ ./build.bash
# ... #
→ Step 4: Process and Copy Info.plist
✅ Copy ExampleApp/Info.plist to _BuildTemp/Info.plist
✅ Set CFBundleExecutable to ExampleApp
✅ Set CFBundleIdentifier to com.vojtastavik.ExampleApp
✅ Set CFBundleName to ExampleApp
✅ Copy _BuildTemp/Info.plist to ExampleApp.app/Info.plist
{% endhighlight %}

We should also verify that the processed `plist` file is there, and its content is correct:

```sh
$ cat ExampleApp.app/Info.plist
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  # ... #
    <key>CFBundleExecutable</key>
    <string>ExampleApp</string>
    <key>CFBundleIdentifier</key>
    <string>com.vojtastavik.ExampleApp</string>
    <key>CFBundleInfoDictionaryVersion</key>
    <string>6.0</string>
    <key>CFBundleName</key>
    <string>ExampleApp</string>
  # ... #
</dict>
</plist>
```

---

## Running ExampleApp in the Simulator

At this point, we should have a valid `.app` bundle. At least for running it in the iOS simulator. You can open the simulator directly from the terminal window:

{% highlight sh %}
$ open -a "Simulator.app"
{% endhighlight %}

To interact with iOS simulators, Apple created a tool named `simctl`. Here's how you can install the `ExampleApp` app bundle to the currently running simulator. The second command starts the installed app.

{% highlight sh %}
$ xcrun simctl install booted ExampleApp.app
{% endhighlight %}

{% highlight sh %}
$ xcrun simctl launch booted com.vojtastavik.ExampleApp
{% endhighlight %}


![Build System](/images/2018-10-15/build-system-5.gif)



##### I must admit I was extremely surprised when I saw the app was working!

The truth is, I thought one more step would be needed. Swift doesn't have a stable binary interface yet and Swift runtime is not included in iOS. Thus every app has to have its own copy of the Swift runtime libraries included in the bundle.

> **We haven't copied the runtime libraries into the bundle but the app works!**
> ![Build System](/images/2018-10-15/build-system-6.jpeg)

When compiling Swift files using `swiftc`, you can specify a `dylib` runtime search folder using `-Xlinker -rpath` flags. By doing so, you're letting the linker know where to search for dynamic libraries, like the Swift-runtime ones. It turns out if you don't specify them *as I did*, the search path defaults to:

```
/Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/lib/swift/iphonesimulator
```

**This is exactly the place where the Swift runtime libraries are stored on your computer!**

The iOS simulator is not sandboxed and has access to all your files. It means it can easily load the runtime libraries from any arbitrary place.  I had no idea about this.

> **As long as you target the iOS simulator, you can create a valid, fully functional, Swift iOS app without including the Swift runtime libs in the bundle.**

![Build System](/images/2018-10-15/build-system-7.gif)

---

## Building for Device

As you can see, building for Simulator is quite forgiving. We don't need to include the runtime libraries in the bundle. *We also don't need to sign it and deal with provisioning.*

> **Changing the build script so it can produce a valid app bundle, which can be executed on an iOS device, is another level of fun.**

#### Firstly, we need to let the script know when to build for device.
Let's use the `--device` flag to signal the script the desired target architecture. We update the code from step 1:

```
#############################################################
#  build.bash
#############################################################

#!/bin/bash

# Exit this script immediately if any of the commands fails
set -e

PROJECT_NAME=ExampleApp

# The product of this script. This is the actual app bundle!
BUNDLE_DIR=${PROJECT_NAME}.app

# A place for any temporary files needed for building
TEMP_DIR=_BuildTemp
```
```sh
# Check for --device flag
if [ "$1" = "--device" ]; then                            #
  BUILDING_FOR_DEVICE=true;                               #
fi                                                        #
                                                          #
# Print the current target architecture
if [ "${BUILDING_FOR_DEVICE}" = true ]; then              #
  echo 👍 Building ${PROJECT_NAME} for device             #
else                                                      #
  echo 👍 Building ${PROJECT_NAME} for simulator          #
fi                                                        #
```
```
#############################################################
echo → Step 1: Prepare Working Folders
#############################################################
...
```

#### Secondly, we need to update the code from step 2:

```
#############################################################
echo → Step 2: Compile Swift Files
#############################################################

# The root directory of the project sources
SOURCE_DIR=ExampleApp

# All Swift files in the source directory
SWIFT_SOURCE_FILES=${SOURCE_DIR}/*.swift
```
```sh
# Target architecture we want to build for
TARGET=""

# Path to the SDK we want to use for compiling
SDK_PATH=""

if [ "${BUILDING_FOR_DEVICE}" = true ]; then
  # Building for device
  TARGET=arm64-apple-ios12.0
  SDK_PATH=$(xcrun --show-sdk-path --sdk iphoneos)

  # The folder inside the app bundle where we
  # will copy all required dylibs
  FRAMEWORKS_DIR=Frameworks

  # Set additional flags for the compiler
  OTHER_FLAGS="-Xlinker -rpath -Xlinker @executable_path/${FRAMEWORKS_DIR}"

else
  # Building for simulator
  TARGET=x86_64-apple-ios12.0-simulator
  SDK_PATH=$(xcrun --show-sdk-path --sdk iphonesimulator)
fi

# Compile sources
swiftc ${SWIFT_SOURCE_FILES} \
  -sdk ${SDK_PATH} \
  -target ${TARGET} \
  -emit-executable \
  ${OTHER_FLAGS} \
  -o ${BUNDLE_DIR}/${PROJECT_NAME}
```
```
echo ✅ Compile Swift source files ${SWIFT_SOURCE_FILES}
```

Changes we made:
- We check the value of `BUILDING_FOR_DEVICE` and set `TARGET` and `SDK_PATH` variables accordingly.
- We introduced the `OTHER_FLAGS` variable. This variable is empty for simulator builds. For device builds, we pass to the compiler additional information about where to find dynamic libraries in the app bundle. **For device builds, we will HAVE TO include Swift runtime in the app bundle.**


#### Finally:
Because all the next steps will be needed only for device builds, if we build for simulator, we can exit the build script right after step 4.

Let's add this to the end of the script:

```sh
#############################################################
if [ "${BUILDING_FOR_DEVICE}" != true ]; then
  # If we build for simulator, we can exit the scrip here
  echo 🎉 Building ${PROJECT_NAME} for simulator successfully finished! 🎉
  exit 0
fi
#############################################################
```


---

## Step 5: Copy Swift Runtime Libraries

I already mentioned, when building for device, we need to include the Swift runtime libraries in the app bundle. Fortunately, no magic is required here. We can just simply copy the libraries to the bundle.

```sh
#############################################################
echo → Step 5: Copy Swift Runtime Libraries
#############################################################

# The folder where the Swift runtime libs live on the computer
SWIFT_LIBS_SRC_DIR=/Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/lib/swift/iphoneos

# The folder inside the app bundle where we want to copy them
SWIFT_LIBS_DEST_DIR=${BUNDLE_DIR}/${FRAMEWORKS_DIR}

# The list of all libs we want to copy
SWIFT_RUNTIME_LIBS=( libswiftCore.dylib libswiftCoreFoundation.dylib libswiftCoreGraphics.dylib libswiftCoreImage.dylib libswiftDarwin.dylib libswiftDispatch.dylib libswiftFoundation.dylib libswiftMetal.dylib libswiftObjectiveC.dylib libswiftQuartzCore.dylib libswiftSwiftOnoneSupport.dylib libswiftUIKit.dylib libswiftos.dylib )

mkdir -p ${BUNDLE_DIR}/${FRAMEWORKS_DIR}
echo ✅ Create ${SWIFT_LIBS_DEST_DIR} folder

for library_name in "${SWIFT_RUNTIME_LIBS[@]}"; do
  # Copy the library
  cp ${SWIFT_LIBS_SRC_DIR}/$library_name ${SWIFT_LIBS_DEST_DIR}/
  echo ✅ Copy $library_name to ${SWIFT_LIBS_DEST_DIR}
done
```

Let's run the script and verify the result. Don't forget to use `--device`.

```
$ ./build.bash --device
👍 Bulding ExampleApp for device
# ... #
→ Step 5: Copy Swift Runtime Libraries
✅ Copy libswiftCore.dylib to ExampleApp.app/Frameworks
✅ Copy libswiftCoreFoundation.dylib to ExampleApp.app/Frameworks
✅ Copy libswiftCoreGraphics.dylib to ExampleApp.app/Frameworks
✅ Copy libswiftCoreImage.dylib to ExampleApp.app/Frameworks
✅ Copy libswiftDarwin.dylib to ExampleApp.app/Frameworks
✅ Copy libswiftDispatch.dylib to ExampleApp.app/Frameworks
✅ Copy libswiftFoundation.dylib to ExampleApp.app/Frameworks
✅ Copy libswiftMetal.dylib to ExampleApp.app/Frameworks
✅ Copy libswiftObjectiveC.dylib to ExampleApp.app/Frameworks
✅ Copy libswiftQuartzCore.dylib to ExampleApp.app/Frameworks
✅ Copy libswiftSwiftOnoneSupport.dylib to ExampleApp.app/Frameworks
✅ Copy libswiftUIKit.dylib to ExampleApp.app/Frameworks
✅ Copy libswiftos.dylib to ExampleApp.app/Frameworks
```
```sh
$ tree -L 2 ExampleApp.app
ExampleApp.app
├── Base.lproj
│   ├── LaunchScreen.storyboardc
│   └── Main.storyboardc
├── ExampleApp
├── Frameworks
│   ├── libswiftCore.dylib
│   ├── libswiftCoreFoundation.dylib
│   ├── libswiftCoreGraphics.dylib
│   ├── libswiftCoreImage.dylib
│   ├── libswiftDarwin.dylib
│   ├── libswiftDispatch.dylib
│   ├── libswiftFoundation.dylib
│   ├── libswiftMetal.dylib
│   ├── libswiftObjectiveC.dylib
│   ├── libswiftQuartzCore.dylib
│   ├── libswiftSwiftOnoneSupport.dylib
│   ├── libswiftUIKit.dylib
│   └── libswiftos.dylib
└── Info.plist
```

---

## Step 6: Code Signing

I spent over 20 hours trying to figure out all the steps needed to successfully build the app for device. A half of this time I spent on this one step.

Before we start, here's a nice picture of kittens for you.

![Build System](/images/2018-10-15/kittens.jpg)

#### 6.1 Provisioning Profile

All your installed provisioning profiles are stored in `~/Library/MobileDevice/Provisioning\ Profiles/`. The first challenge is to find the correct one to use for this app.

**I'd strongly recommend to figure everything out in Xcode, first.** If I'm not mistaken, with the free Apple dev account, you can't even create a new provisioning profile using the web portal. You have to use Xcode for that.

Once you have the code signing working there, make a note of which provisioning profile and signing identity were used.

For the app bundle to be valid, it has to contain a file named `embedded.mobileprovision` in its root. Let's copy the correct provisioning profile to the app bundle, and rename it:

```sh
#############################################################
echo → Step 6: Code Signing
#############################################################

# The name of the provisioning file to use
# ⚠️ YOU NEED TO CHANGE THIS TO YOUR PROFILE ️️⚠️
PROVISIONING_PROFILE_NAME=23a6e9d9-ad3c-4574-832c-be6eb9d51b8c.mobileprovision

# The location of the provisioning file inside the app bundle
EMBEDDED_PROVISIONING_PROFILE=${BUNDLE_DIR}/embedded.mobileprovision

cp ~/Library/MobileDevice/Provisioning\ Profiles/${PROVISIONING_PROFILE_NAME} ${EMBEDDED_PROVISIONING_PROFILE}
echo ✅ Copy provisioning profile ${PROVISIONING_PROFILE_NAME} to ${EMBEDDED_PROVISIONING_PROFILE}
```


#### 6.2 Signing Entitlements

Another file needed to successfully sign the bundle is the `.xcent` file. This file is just another `plist`, created by merging project's `Entitlements` file with additional signing information.

There's no `Entitlements` file in our project, so this part of _merging_ is done 👍.

The additional signing information, I was referring to, is the Apple dev team ID. **This ID has to be the one from the signing identity you are about to use in the next step.**

Let's create the `.xcent` file and set the required values:

```
#############################################################
echo → Step 6: Code Signing
#############################################################

...

```
```sh
# The team identifier of your signing identity
# ⚠️ YOU NEED TO CHANGE THIS TO YOUR ID ️️⚠️
TEAM_IDENTIFIER=X53G3KMVA6

# The location if the .xcent file
XCENT_FILE=${TEMP_DIR}/${PROJECT_NAME}.xcent

# The file doesn't exist but PlistBuddy will create it automatically
${PLIST_BUDDY} -c "Add :application-identifier string ${TEAM_IDENTIFIER}.${APP_BUNDLE_IDENTIFIER}" ${XCENT_FILE}
${PLIST_BUDDY} -c "Add :com.apple.developer.team-identifier string ${TEAM_IDENTIFIER}" ${XCENT_FILE}

echo ✅ Create ${XCENT_FILE}
```


#### 6.3 Signing

Now, when we have all the required pieces in place, we perform the actual signing. The tool which will help us with this is called `codesign`.

You need to specify which identity should be used for signing. You can list all available identities in the terminal using:
```
$ security find-identity -v -p codesigning
```

For the app bundle to be valid, we need to sign all dynamic libraries in the `Frameworks` folder, and then the bundle itself. We use the `xcent` file only for the final signing. **Let's finish this!**

```
#############################################################
echo → Step 6: Code Signing
#############################################################

...

```
```sh
# The id of the identity used for signing
IDENTITY=E8C36646D64DA3566CB93E918D2F0B7558E78BAA

# Sign all libraries in the bundle
for lib in ${SWIFT_LIBS_DEST_DIR}/*; do
  # Sign
  codesign \
    --force \
    --timestamp=none \
    --sign ${IDENTITY} \
    ${lib}
  echo ✅ Codesign ${lib}
done

# Sign the bundle itself
codesign \
  --force \
  --timestamp=none \
  --sign ${IDENTITY} \
  --entitlements ${XCENT_FILE} \
  ${BUNDLE_DIR}


echo ✅ Codesign ${BUNDLE_DIR}

```

#### Celebrate!

The last part of the script we need to write is the final message:

```sh
#############################################################
echo 🎉 Building ${PROJECT_NAME} for device successfully finished! 🎉
exit 0
#############################################################
```

You can now run the script, sit back, and enjoy your victory:
```
$ ./build.bash --device
👍 Building ExampleApp for device

→ Step 1: Prepare Working Folders
✅ Create ExampleApp.app folder
✅ Create _BuildTemp folder

→ Step 2: Compile Swift Files
✅ Compile Swift source files ExampleApp/AppDelegate.swift ExampleApp/ViewController.swift

→ Step 3: Compile Storyboards
✅ Create ExampleApp.app/Base.lproj folder
✅ Compile ExampleApp/Base.lproj/LaunchScreen.storyboard
✅ Compile ExampleApp/Base.lproj/Main.storyboard

→ Step 4: Process and Copy Info.plist
✅ Copy ExampleApp/Info.plist to _BuildTemp/Info.plist
✅ Set CFBundleExecutable to ExampleApp
✅ Set CFBundleIdentifier to com.vojtastavik.ExampleApp
✅ Set CFBundleName to ExampleApp
✅ Copy _BuildTemp/Info.plist to ExampleApp.app/Info.plist

→ Step 5: Copy Swift Runtime Libraries
✅ Create ExampleApp.app/Frameworks folder
✅ Copy libswiftCore.dylib to ExampleApp.app/Frameworks
✅ Copy libswiftCoreFoundation.dylib to ExampleApp.app/Frameworks
✅ Copy libswiftCoreGraphics.dylib to ExampleApp.app/Frameworks
✅ Copy libswiftCoreImage.dylib to ExampleApp.app/Frameworks
✅ Copy libswiftDarwin.dylib to ExampleApp.app/Frameworks
✅ Copy libswiftDispatch.dylib to ExampleApp.app/Frameworks
✅ Copy libswiftFoundation.dylib to ExampleApp.app/Frameworks
✅ Copy libswiftMetal.dylib to ExampleApp.app/Frameworks
✅ Copy libswiftObjectiveC.dylib to ExampleApp.app/Frameworks
✅ Copy libswiftQuartzCore.dylib to ExampleApp.app/Frameworks
✅ Copy libswiftSwiftOnoneSupport.dylib to ExampleApp.app/Frameworks
✅ Copy libswiftUIKit.dylib to ExampleApp.app/Frameworks
✅ Copy libswiftos.dylib to ExampleApp.app/Frameworks

→ Step 6: Code Signing
✅ Copy provisioning profile 23a6e9d9-ad3c-4574-832c-be6eb9d51b8c.mobileprovision to ExampleApp.app/embedded.mobileprovision
File Doesn't Exist, Will Create: _BuildTemp/ExampleApp.xcent
✅ Create _BuildTemp/ExampleApp.xcent
ExampleApp.app/Frameworks/libswiftCore.dylib: replacing existing signature
✅ Codesign ExampleApp.app/Frameworks/libswiftCore.dylib
ExampleApp.app/Frameworks/libswiftCoreFoundation.dylib: replacing existing signature
✅ Codesign ExampleApp.app/Frameworks/libswiftCoreFoundation.dylib
ExampleApp.app/Frameworks/libswiftCoreGraphics.dylib: replacing existing signature
✅ Codesign ExampleApp.app/Frameworks/libswiftCoreGraphics.dylib
ExampleApp.app/Frameworks/libswiftCoreImage.dylib: replacing existing signature
✅ Codesign ExampleApp.app/Frameworks/libswiftCoreImage.dylib
ExampleApp.app/Frameworks/libswiftDarwin.dylib: replacing existing signature
✅ Codesign ExampleApp.app/Frameworks/libswiftDarwin.dylib
ExampleApp.app/Frameworks/libswiftDispatch.dylib: replacing existing signature
✅ Codesign ExampleApp.app/Frameworks/libswiftDispatch.dylib
ExampleApp.app/Frameworks/libswiftFoundation.dylib: replacing existing signature
✅ Codesign ExampleApp.app/Frameworks/libswiftFoundation.dylib
ExampleApp.app/Frameworks/libswiftMetal.dylib: replacing existing signature
✅ Codesign ExampleApp.app/Frameworks/libswiftMetal.dylib
ExampleApp.app/Frameworks/libswiftObjectiveC.dylib: replacing existing signature
✅ Codesign ExampleApp.app/Frameworks/libswiftObjectiveC.dylib
ExampleApp.app/Frameworks/libswiftQuartzCore.dylib: replacing existing signature
✅ Codesign ExampleApp.app/Frameworks/libswiftQuartzCore.dylib
ExampleApp.app/Frameworks/libswiftSwiftOnoneSupport.dylib: replacing existing signature
✅ Codesign ExampleApp.app/Frameworks/libswiftSwiftOnoneSupport.dylib
ExampleApp.app/Frameworks/libswiftUIKit.dylib: replacing existing signature
✅ Codesign ExampleApp.app/Frameworks/libswiftUIKit.dylib
ExampleApp.app/Frameworks/libswiftos.dylib: replacing existing signature
✅ Codesign ExampleApp.app/Frameworks/libswiftos.dylib
✅ Codesign ExampleApp.app

🎉 Building ExampleApp for device successfully finished! 🎉

```

---

## Installing to an iOS Device

Unfortunately, installing the app to a connected iOS device is not at all that straightforward as it was for the simulator. Luckily, there are 3rd party tools which make the task much more pleasant.

**I'm using [ios-deploy](https://github.com/ios-control/ios-deploy).**

You can list all connected devices using `-c` and then copy the identifier of the device.
```
$ ios-deploy -c
[....] Waiting up to 5 seconds for iOS device to be connected
[....] Found 00008020-xxxxxxxxxxxx (D321AP, D321AP, uknownos, unkarch) a.k.a. 'Vojta's iPhone' connected through USB.
```

Finally, you install the app bundle to the device using:
```
$ ios-deploy -i 00008020-xxxxxxxxxxxx -b ExampleApp.app
```

![Build System](/images/2018-10-15/build-system-8.gif)

---

## Next Steps

> **Here's the [GitHub repo](https://github.com/VojtaStavik/ios-build-script) with the final version of the script.**

I learned **a lot** when writing this article. I don't personally plan to work on this script anymore. However, if someone is interested in improving it, here are some ideas:

- I didn't need to touch `*.xcassets` files because there was no content inside. However, these files also need to be compiled.
- It would be nice to get incremental builds working for Swift.
- The script currently builds only Swift files. What about projects with Objective-C, C++, Objective-C++, C sources?
- I don't think it would make sense to parse `xcodeproj` and get the build parameters from it. What about getting the build settings from an [XcodeGen](https://github.com/yonaskolb/XcodeGen) file?
- What about target dependencies (like custom embedded frameworks) that also need to be built?

---

*Special thanks to [Harlan Haskins](https://twitter.com/harlanhaskins) for helping me with the article by answering my questions.*
