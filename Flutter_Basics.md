Below is an updated, complete Markdown tutorial that not only shows you how to create your first Flutter app with Visual Studio Code (VS Code) and Ubuntu WSL on Windows 11, but also includes the steps for installing the VS Code Remote – WSL extension, setting up an Android emulator, and (if needed) integrating Kotlin code into your app.

---

# Flutter App Tutorial: From Environment Setup to Kotlin Integration

This guide assumes you are new to Dart and Flutter. You have already installed VS Code and Ubuntu WSL on your Windows 11 system. Follow these steps to set up your Flutter development environment, install the VS Code Remote – WSL extension, configure an Android emulator, create your first Flutter app, and (if necessary) integrate Kotlin code into your app.

---

## 1. Setting Up Your Flutter Environment

### a. Download the Flutter SDK
1. Open your Ubuntu WSL terminal.
2. Run the following commands to clone the stable branch of the Flutter SDK:
   ```bash
   cd ~
   git clone https://github.com/flutter/flutter.git -b stable
   ```
   This clones Flutter into a `flutter` directory in your home folder.

### b. Update Your PATH
1. Open your `~/.bashrc` file:
   ```bash
   nano ~/.bashrc
   ```
2. Add the following line at the end of the file:
   ```bash
   export PATH="$PATH:$HOME/flutter/bin"
   ```
3. Save (Ctrl+O) and exit (Ctrl+X).
4. Reload your shell:
   ```bash
   source ~/.bashrc
   ```

### c. Verify the Installation
Run:
```bash
flutter doctor
```
Flutter Doctor will check your environment and list any missing dependencies (e.g., Android SDK). Follow its instructions to install any necessary components.

---

## 2. Installing VS Code Remote – WSL Extension

### a. Open VS Code
1. Launch VS Code on your Windows 11 system.

### b. Install the Extension
1. Click on the **Extensions** icon in the Activity Bar on the side of VS Code (or press `Ctrl+Shift+X`).
2. In the search bar, type **Remote - WSL**.
3. Click **Install** on the **Remote - WSL** extension provided by Microsoft.

### c. Open a WSL Window
1. Once installed, press `Ctrl+Shift+P` to open the Command Palette.
2. Type **Remote-WSL: New Window** and select it. This will open a new VS Code window connected to your Ubuntu WSL environment.
3. You can now open your project folder from WSL by using:
   ```bash
   code .
   ```

---

## 3. Creating a New Flutter Project

### a. Use the Flutter CLI
1. In your Ubuntu WSL terminal (or the integrated terminal in VS Code running WSL), navigate to your projects directory:
   ```bash
   cd ~/projects
   ```
2. Create a new Flutter project:
   ```bash
   flutter create my_first_flutter_app
   ```
3. Change directory into your new project:
   ```bash
   cd my_first_flutter_app
   ```

### b. Open the Project in VS Code
In your terminal, type:
```bash
code .
```
This opens your project in the VS Code window that is connected to WSL.

---

## 4. Exploring and Modifying the Example App

### a. Understanding the Project Structure
- **lib/main.dart:** The main entry point of your Flutter app.
- **pubspec.yaml:** Contains your app’s metadata and dependency settings.
- **assets & test folders:** Used for adding assets (images, fonts, etc.) and tests.

### b. Review the Main Code
Open `lib/main.dart` and you will see code similar to:
```dart
import 'package:flutter/material.dart';

void main() => runApp(MyApp());

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter Demo',
      home: Scaffold(
        appBar: AppBar(
          title: Text('My First Flutter App'),
        ),
        body: Center(
          child: Text('Hello, Flutter!'),
        ),
      ),
    );
  }
}
```
This simple code creates a Material Design app with an AppBar and a centered text.

### c. Customize the App
- Modify the text within the `Text` widget.
- Change the `title` property to your desired app name.
- Experiment by adding more widgets such as buttons or images to explore Flutter’s UI components.

---

## 5. Installing and Configuring an Android Emulator

### a. Install Android Studio (Windows)
1. Download and install [Android Studio](https://developer.android.com/studio) on your Windows system.
2. During installation, ensure you install the Android SDK, Android Virtual Device (AVD) Manager, and necessary drivers.

### b. Create an Android Virtual Device (AVD)
1. Open Android Studio.
2. Click on **Configure** (or find the AVD Manager from the welcome screen).
3. Select **AVD Manager** and click **Create Virtual Device**.
4. Choose a device definition (e.g., Pixel 4) and click **Next**.
5. Choose a system image (make sure it is a supported one, like one based on Android 10 or higher) and click **Next**.
6. Adjust AVD settings as needed and click **Finish**.

### c. Run the Emulator
- Start your AVD from the AVD Manager.
- Flutter should automatically detect the running emulator when you run `flutter run` or select it in VS Code.

---

## 6. Running Your Flutter App

### a. Select a Device
1. With your Android emulator running (or using Flutter Web if you prefer), open the Command Palette in VS Code (`Ctrl+Shift+P`).
2. Type **Flutter: Select Device** and choose your target (emulator, web, etc.).

### b. Launch the App
Run your app by either:
- Pressing `F5` in VS Code, or
- Running the command:
  ```bash
  flutter run
  ```
Your app should now run on the selected device/emulator.

---

## 7. Integrating Kotlin Code into Your Flutter App

If you need to integrate custom Kotlin code (for Android-specific functionality), follow these steps:

### a. Locate the Android Directory
- In your Flutter project, navigate to the `android/` folder.
- The Kotlin code usually resides under `android/app/src/main/kotlin/`.  
  If the folder structure uses Java, you can convert the project to Kotlin.

### b. Convert to Kotlin (if necessary)
1. Open the `android` folder in Android Studio.
2. If your project is in Java, Android Studio may prompt you to convert to Kotlin. Follow the instructions to migrate the project.
3. Alternatively, manually create a Kotlin file inside the appropriate package directory (e.g., `com/example/my_first_flutter_app`).

### c. Add Kotlin Dependencies
1. In `android/build.gradle`, ensure you have a Kotlin version defined. For example:
   ```gradle
   buildscript {
       ext.kotlin_version = '1.7.10'
       repositories {
           google()
           mavenCentral()
       }
       dependencies {
           classpath 'com.android.tools.build:gradle:7.0.2'
           classpath "org.jetbrains.kotlin:kotlin-gradle-plugin:$kotlin_version"
       }
   }
   ```
2. In `android/app/build.gradle`, apply the Kotlin Android plugin at the top:
   ```gradle
   apply plugin: 'com.android.application'
   apply plugin: 'kotlin-android'
   ```

### d. Write and Integrate Your Kotlin Code
- Create your Kotlin file (for example, `CustomFeature.kt`) in the `kotlin` directory.
- Write your custom Android code in Kotlin.
- To call Kotlin code from Flutter, you’ll need to use platform channels. In your Flutter app, define a method channel:
  ```dart
  import 'package:flutter/services.dart';

  class CustomPlatform {
    static const platform = MethodChannel('com.example.my_first_flutter_app/custom');

    static Future<String> getCustomData() async {
      try {
        final String result = await platform.invokeMethod('getCustomData');
        return result;
      } on PlatformException catch (e) {
        return "Failed: '${e.message}'.";
      }
    }
  }
  ```
- In your Kotlin file, set up a `MethodChannel` in the `MainActivity` (or your designated activity) to listen for calls from Flutter:
  ```kotlin
  package com.example.my_first_flutter_app

  import io.flutter.embedding.android.FlutterActivity
  import io.flutter.embedding.engine.FlutterEngine
  import io.flutter.plugin.common.MethodChannel

  class MainActivity: FlutterActivity() {
      private val CHANNEL = "com.example.my_first_flutter_app/custom"

      override fun configureFlutterEngine(flutterEngine: FlutterEngine) {
          super.configureFlutterEngine(flutterEngine)
          MethodChannel(flutterEngine.dartExecutor.binaryMessenger, CHANNEL).setMethodCallHandler { call, result ->
              if (call.method == "getCustomData") {
                  // Add your custom Kotlin code here.
                  val data = "Hello from Kotlin!"
                  result.success(data)
              } else {
                  result.notImplemented()
              }
          }
      }
  }
  ```
- Now, calling `CustomPlatform.getCustomData()` in Flutter will trigger your Kotlin code and return the custom message.

---

## Conclusion

You now have a full guide to set up your Flutter development environment on Windows 11 with VS Code and Ubuntu WSL. You learned how to:

- Install the Flutter SDK and update your PATH.
- Install the VS Code Remote – WSL extension.
- Create a new Flutter project.
- Set up an Android emulator using Android Studio.
- Run your Flutter app.
- Integrate and call Kotlin code from Flutter using platform channels.

Happy coding and enjoy building your Flutter app!