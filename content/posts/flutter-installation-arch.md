---
title: "Installing Flutter and Android SKD on Arch Linux without Android Studio"
date: 2023-05-30T18:33:05+02:00
tags: ["Flutter"]
---

Having recently switched from Arcolinux to Arch Linux, I needed to reinstall Flutter on my laptop. The last time I did this, I installed Android Studio because it seemed like the easiest way to get the Android SDK working. However, I never used it as an IDE, so it was hard to justify keeping it on my system since it takes up more than 30GB.

This time, with a year of Linux experience under my belt, I've managed to install and configure everything needed to build apps for Android without having to install Android Studio. In this article, I'll tell you how I did it.

The first step is to install `dart` from the "extra" repository and `flutter` from the AUR. Flutter depends on the Java JDK, so you have to choose which version to install. At first I went with the default option (jdk19), but then I ran into compatibility issues and had to switch to `jdk11-openjdk`. Later versions are not yet [supported](https://docs.gradle.org/current/userguide/compatibility.html#kotlin) by the version of the [gradle plugin](https://mvnrepository.com/artifact/com.android.tools.build/gradle?repo=google) that Flutter uses to build Android apps.

When the installation was over, I tried to run `flutter doctor`, but got this error

```
fatal: detected dubious ownership in repository at '/opt/flutter'
To add an exception for this directory, call:
git config --global --add safe.directory /opt/flutter
```

I ran the recommended command to mark the `/opt/flutter` directory as a safe git repository. Then, I tried again and got the following message

```
Flutter failed to create a directory at "/opt/flutter/bin/cache/downloads".
Please ensure that the SDK and/or project is installed in a location that has read/write permissions for the current user.
```

To fix this, I added my user to the `flutterusers` group (created automatically during installation) to have read and write access to the Flutter SDK. This was recommended in the message I got at the end of the Flutter installation.

```
sudo gpasswd -a $USER flutterusers
newgrp flutterusers
```

The `newgrp` command lets you login your terminal in to the newly added group.

Finally, I managed to run `flutter doctor` and got this output

```
Doctor summary (to see all details, run flutter doctor -v):
[!] Flutter (Channel stable, 3.10.0, on Arch Linux 6.3.3-arch1-1, locale C.UTF-8)
	! Warning: `dart` on your path resolves to /opt/dart-sdk/bin/dart, which is not inside your current Flutter SDK checkout at /opt/flutter. Consider adding /opt/flutter/bin to the front of your path.

[✗] Android toolchain - develop for Android devices
    ✗ Unable to locate Android SDK.
      Install Android Studio from: https://developer.android.com/studio/index.html
      On first launch it will assist you in installing the Android SDK components.
      (or visit https://flutter.dev/docs/get-started/install/linux#android-setup for detailed instructions).
      If the Android SDK has been installed to a custom location, please use
      `flutter config --android-sdk` to update to that location.

[✗] Chrome - develop for the web (Cannot find Chrome executable at google-chrome)
    ! Cannot find Chrome. Try setting CHROME_EXECUTABLE to a Chrome executable.
[✗] Linux toolchain - develop for Linux desktop
    ✗ CMake is required for Linux development.
      It is likely available from your distribution (e.g.: apt install cmake), or can be downloaded from https://cmake.org/download/
    ✗ ninja is required for Linux development.
      It is likely available from your distribution (e.g.: apt install ninja-build), or can be downloaded from https://github.com/ninja-build/ninja/releases
[!] Android Studio (not installed)
[✓] Connected device (1 available)
[✓] HTTP Host Availability

! Doctor found issues in 5 categories.
```

To develop for Android, only the warning and the Android toolchain issue need to be addressed, the other can be ignored.

## Dart not inside your current Flutter SDK

The warning is pretty self-explanatory

```
! Warning: `dart` on your path resolves to /opt/dart-sdk/bin/dart, which is not inside your current Flutter SDK at /opt/flutter. Consider adding /opt/flutter/bin to the front of your path.
```

To fix this, you simply need to add `/opt/flutter/bin` to the front of your `PATH`. You can do this by adding the following line to the shell configuration file of your choice (eg. `.bashrc`, `.zshrc`). Personally, I export all environment variables from `.zshenv`.

```bash
export PATH=/opt/flutter/bin:$PATH
```

## Install the Android toolchain

To have access to the Andoid toolchain in its most minimal form, install the following packages from the AUR:
- `android-sdk`: Google Android SDK
- `android-sdk-cmdline-tools-latest`: Android SDK Command-line Tools
- `android-sdk-platform-tools`: Platform-Tools for Google Android SDK (adb and fastboot)
- `android-sdk-build-tools`: Build-Tools for Google Android SDK (aapt, aidl, dexdump, dx, llvm-rs-cc)

Command line tools are located in `/opt/android-sdk/cmdline-tools/latest/bin`, so you can add this folder to your `PATH` to access them from anywhere. Otherwise, you'll need to use the full path to each tool.

To setup the Android SDK, export the following environment variables like you did for the `PATH`.

```bash
export ANDROID_HOME=/opt/android-sdk
export JAVA_HOME='/usr/lib/jvm/java-10-openjdk'
```

Then, create an `android-sdk` group, give it access to the folder where the SDK is stored and add add your user to it.

```bash
sudo groupadd android-sdk
sudo setfacl -R -m g:android-sdk:rwx /opt/android-sdk
sudo setfacl -d -m g:android-sdk:rwX /opt/android-sdk
sudo gpasswd -a $USER android-sdk
```

If `flutter doctor` still complains about not being able to find the Android SKD try running

```bash
flutter config --android-sdk $ANDROID_SDK_ROOT
```

Before you can build an Android app, you need to install a platform using `sdkmanager`.

```bash
sdkmanager --install "platforms;android-33"
sdkmanager --list
```

Finally, accept all licenses with the following command.

```bash
flutter doctor --android-licenses
```

Personally, I run my apps on a physical device connected to my laptop in debug mode, instead of using an emulator. However, if you want to install an emulator, you can do so with `avdmanager`. You can read more about it in the [official documentation](https://developer.android.com/tools/avdmanager).

If you've followed along with all the steps, you should now be able to build Flutter apps for Android.

## Tips for throubleshooting

If you encounter any problems during the installation process, I recommend running `flutter doctor` in verbose mode by adding the `-v` flag. This will give you additional information about each issue and probably some instructions on how to fix it.

Also, when in doubt, try rebooting to make sure the new permissions have taken effect.

