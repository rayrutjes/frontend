<!--

title: Test Android applications
short_title: Android

-->

# Android on CircleCI

CircleCI supports building and testing Android applications.

### Dependencies

The SDK is already installed on the VM at `/usr/local/android-sdk-linux`. We export
this path as `$ANDROID_HOME`.

We have the following SDK packages preinstalled:

{{ versions.android_sdk_packages | code-list}} 

If there's an SDK package that's not here that you would like
installed, you can install it as part of your build with:

```
dependencies:
  pre:
    - echo y | android update sdk --no-ui --all --filter "package-name"
```

We're currently missing the Google API addons, but these will come pre-installed in a future release. In the meantime, you can install them like so:
```
dependencies:
  pre:
    - echo y | android update sdk --no-ui --all --filter sys-img-armeabi-v7a-addon-google_apis-google-21,addon-google_apis-google-21
```

We also preinstall the Android NDK; it can be found at `$ANDROID_NDK`.

`./gradlew dependencies` will also be run automatically if you have a
Gradle wrapper checked in to the root of your repository.

### Building Android Projects Manually

If you only want to build your project you can create a debug build with

```
test:
  override:
    - ./gradlew assembleDebug
```

or build a release `.apk` and save it to [artifacts](build-artifacts) with

```
test:
  override:
    - ./gradlew assembleRelease
    - cp -r project-name/build/outputs $CIRCLE_ARTIFACTS
```

If you start the emulator, you can install your APK on it with something lik
the following:

```
test:
  override:
    - adb install path/to/build.apk
```


#### Disable Pre-Dexing to Improve Build Performance

By default the Gradle android plugin pre-dexes dependencies,
converting their Java bytecode into Android bytecode. This speeds up
development greatly since gradle only needs to do incremental dexing
as you change code.

Because CircleCI always runs clean builds this pre-dexing has no
benefit; in fact it makes compilation slower and can also use large
quantities of memory.  We recommend
[disabling pre-dexing][disable-pre-dexing] for Android builds on
CircleCI.

[disable-pre-dexing]: http://tools.android.com/tech-docs/new-build-system/tips#TOC-Improving-Build-Server-performance

### Testing Android Projects

Firstly: if you have a Gradle wrapper in the root of your repository,
we'll automatically run `./gradlew test`.

<h4 id="emulator">Starting the Android Emulator</h3>

Starting the android emulator can be an involved process and, unfortunately, can take
a few minutes. You can start the emulator and wait for it to finish with something like
the following:

```
test:
  pre:
    - emulator -avd circleci-android22 -no-audio -no-window:
        background: true
        parallel: true
    - circle-android wait-for-boot
```

`circleci-android22` is an AVD preinstalled on the machine for Android 22 on the ARM V7 EABI.
There's also a corresponding `circleci-android21`; alternatively, you can 
[can create your own][create-avd] if these don't suit your purposes.

[create-avd]: https://developer.android.com/tools/devices/managing-avds-cmdline.html#AVDCmdLine

One important note: it's not possible to emulate Android on x86 or
x86_64 on our build containers. The Android emulator requires KVM on
Linux, and we can't provide it.

`circle-android wait-for-boot` is a tool on our build containers that waits for the emulator
to have finished booting. `adb wait-for-device` is not sufficient here; it only waits
for the device's shell to be available, not for the boot process to finish. You can read more about
this [here][starting-emulator].

[starting-emulator]:https://devmaze.wordpress.com/2011/12/12/starting-and-stopping-android-emulators/


#### Running Tests Against the Emulator

The standard way to run tests in the Android emulator is with
something like `./gradlew connectedAndroidTest`.

You may also want to run commands directly with `adb shell`, after
installing your APK on the emulator. Note however that `adb shell`
[does not correctly forward exit codes][adb-shell-bug]. We provide
Facebook's [`fb-adb`][fb-adb] tool on our container images to work
around this: `fb-adb shell` *does* correctly report exit codes. You
should prefer `fb-adb shell` over `adb shell` in CircleCI builds in
order to prevent failing commands from being understood as passing.

[adb-shell-bug]: https://code.google.com/p/android/issues/detail?id=3254
[fb-adb]:https://github.com/facebook/fb-adb


#### Test Metadata

Many test suites for Android produce JUnit XML output. After running your tests,
you can copy that output to `$CIRCLE_TEST_REPORTS` so that CircleCI will display
the individual test results.

#### Deploying to Google Play Store

There are a few plugins for Gradle that allow you to push your `apk` to
Google Play Store with a simple Gradle command, for example [this plugin](https://github.com/Triple-T/gradle-play-publisher).

After applying the plugin and setting up all the configuration details,
you can use the `deployment` section of your `circle.yml` to publish the
`apk` to the desired channel. We suggest reading the channel from
a property in the plugin configuration like this:

```
play {
  track = "${track}"
}
```
This will allow you to specify different deployment channels right in
the `circle.yml`:

```
deployment:
  production: # just a label; label names are completely up to you
    branch: master
    commands:
      - ./gradlew publishApkRelease
        -Dorg.gradle.project.track=production
  beta:
    branch: develop
    commands:
      - ./gradlew publishApkRelease
        -Dorg.gradle.project.track=beta
```

### Sample circle.yml

```
test:
  override:
    # start the emulator
    - emulator -avd circleci-android22 -no-audio -no-window:
        background: true
        parallel: true
    # wait for it to have booted
    - circle-android wait-for-boot
    # run tests  against the emulator.
    - ./gradlew connectedAndroidTest
    # copy the build outputs to artifacts
    - cp -r my-project/build/outputs $CIRCLE_ARTIFACTS
    # copy the test results to the test results directory.
    - cp -r my-project/build/outputs/androidTest-results/* $CIRCLE_TEST_REPORTS
```

Please don't hesitate to [contact us](mailto:sayhi@circleci.com)
if you have any questions at all about how to best test Android on
CircleCI.
