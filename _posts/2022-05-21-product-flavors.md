---
author: Biplab
title: Product Flavors in Flutter—Create admin and non-admin apps with distinct UI with a single codebase
date: 2022-05-21 16:00:00 +545
categories: [flutter]
tags: [flutter, dart, product flavors, build variants]
image:
  path: https://raw.githubusercontent.com/Biplab-Dutta/personal_site/main/preview_images/product_flavor.png
  alt: Preview Image Credit - Glitch
---

Have you ever wondered how some mobile applications have admin and non-admin variants? The admin app has different UIs than the non-admin ones. Or have you seen some apps on the Play Store or App Store with premium and freemium versions? So, how do developers actually do it? How do they create multiple variants of the same project? Do they manage multiple codebases? Is there one team responsible for developing one variant and another team developing the other variant with 2 different codebases? And a clear and short answer to that is **NO**.

It would be costly for companies to hire two different teams to create two different app variants. So, how is it possible? And unsurprisingly, the answer to that is using **[Product Flavor](https://developer.android.com/studio/build/build-variants#product-flavors)**.

As the name suggests, a product flavor (or a product variant) is a way to create multiple variants of your app from a single codebase. We can deploy these different apps independently in the relevant stores as well.

<!-- omit in toc -->
## Table of Contents

<!-- Chirpy theme by default provides a table of contents for desktop users. I am creating one explicitly for mobile users. -->

- [Implementation](#implementation)
  - [For VS Code Users](#for-vs-code-users)
  - [For Android Studio Users](#for-android-studio-users)
- [Conclusion](#conclusion)
- [Credit](#credit)

## Implementation

Now, we will begin creating our flavors. We will have an admin flavor and a non-admin flavor. I will keep the apps very simple and have them display a text saying **`This is the admin UI`** and **`This is the non-admin UI`**. In a real-world application, you can follow the same techniques that I will show you and have UIs accordingly the way you want.

First, we will add a configuration in the app-level `build.gradle` file inside the `android` block.

```groovy
android {
    ...
    defaultConfig {...}
    buildTypes {
        debug{...}
        release{...}
    }
    
    // Specifies one flavor dimension.
    flavorDimensions "userTypes"
    productFlavors {
        admin {
            dimension "userTypes"
            resValue "string", "app_name", "Admin App"
            applicationIdSuffix ".admin"
        }
        non_admin {
            dimension "userTypes"
            resValue "string", "app_name", "Non-Admin App"
            applicationIdSuffix ".nonAdmin"
        }
    }
}

```

Because we will have two different apps created, we want two different names for each of our applications. To do so, we will have to navigate to `/android/app/src/main/AndroidManifest.xml` file and edit `android:label`.

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.product_flavor_demo">
   <application
        android:label="@string/app_name"      <!-- Edit this line -->
        android:name="${applicationName}"
        android:icon="@mipmap/ic_launcher">
        <activity
            android:name=".MainActivity"
            android:exported="true"
            android:launchMode="singleTop"
            android:theme="@style/LaunchTheme"
            android:configChanges="orientation|keyboardHidden|keyboard|screenSize|smallestScreenSize|locale|layoutDirection|fontScale|screenLayout|density|uiMode"
            android:hardwareAccelerated="true"
            android:windowSoftInputMode="adjustResize">
            <!-- Specifies an Android theme to apply to this Activity as soon as
                 the Android process has started. This theme is visible to the user
                 while the Flutter UI initializes. After that, this theme continues
                 to determine the Window background behind the Flutter UI. -->
            <meta-data
              android:name="io.flutter.embedding.android.NormalTheme"
              android:resource="@style/NormalTheme"
              />
            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>
        <!-- Don't delete the meta-data below.
             This is used by the Flutter tool to generate GeneratedPluginRegistrant.java -->
        <meta-data
            android:name="flutterEmbedding"
            android:value="2" />
    </application>
</manifest>
```

Now, we need to create two `main.dart` files in our `lib` directory. We shall name them `main_admin.dart` and `main_non_admin.dart`.

```dart
// main_admin.dart

// Add necessary imports

void main() {
  runApp(const MyApp());
}
```

```dart
// main_non_admin.dart

// Add necessary imports

void main() {
  runApp(const MyApp());
}
```

We will create our `MyApp()` widget in a moment but let’s first take care of some other things.

### For VS Code Users

If you are a VS Code user, then you need to follow some of the steps that I’ll show you now.

First, create a `.vscode` folder in the root project directory. Then create a file `launch.json` inside it and add the following snippet.

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "admin_app",
            "request": "launch",
            "type": "dart",
            "program": "lib/main_admin.dart",
            "args": [
                "--flavor",
                "admin",
                "--target",
                "lib/main_admin.dart",
                "--dart-define=appType=admin"
            ]
        },
        {
            "name": "non_admin_app",
            "request": "launch",
            "type": "dart",
            "program": "lib/main_non_admin.dart",
            "args": [
                "--flavor",
                "non_admin",
                "--target",
                "lib/main_non_admin.dart",
                "--dart-define=appType=nonAdmin"
            ]
        }
    ]
}
```

Now, if you go to the `Run and Debug` option in your VS Code or hold <kbd>Ctrl</kbd>+<kbd>Shift</kbd>+<kbd>D</kbd>, you will see a drop-down menu. On clicking it, you should see an option to debug your two different app variants.

![vs code product flavor screenshot](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/zo2s4i4u8411fra9bik7.png)

### For Android Studio Users

If you use Android Studio then you need to follow some of the steps that I’ll show you now.

Navigate to `Edit Configurations` option under the `Run` tab. It should open up a new window. Then you need to add configurations for each flavor.

![Debug Configuration Window Android Studio](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/i76t2x5afzwcnuwkbd11.png)

In the `Dart entrypoint` option, add the path to `main_admin.dart` file using the browse option on the right-hand side. In the `Additional run args` option, add

```
--flavor admin --dart-define=appType=admin
```

Now, add another configuration for the non-admin app.

![Debug Configuration Window Android Studio](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/g60fheqaniw2t1wjlbii.png)

Follow the same steps as mentioned above and in the `Additional run args` option, add

```
--flavor non_admin --dart-define=appType=nonAdmin
```

Now, we can select the proper configurations that we want to run and debug.

![Debugger Android Studio](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/h2puhuqa5977gfuafsjy.png)

The `dart-define` option that we have attached in our command is important to find out the app type on run time. We will see how we can use it to identify the app types.

Create a new file `app_config.dart` inside the lib directory.

```dart
abstract class AppConfig {
  static const isAdminApp = String.fromEnvironment('appType') == 'admin';
}
```

The value of `String.fromEnvironment()` comes from the `dart-define` option that we set earlier for each app variant. Now, using the `isAdminApp` boolean value, we can easily check if the app running currently is the admin app or the non-admin app and render UIs accordingly.

Now create a new file `my_app.dart` inside the `lib` directory which will contain code for our `MyApp()` class. I am keeping it very simple to display different UI for each app variant. You can however take the idea and create as complex UI as you want for each app variant.

```dart
// Add the necessary imports

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: AppConfig.isAdminApp ? 'Admin App' : 'Non-admin App',
      theme: ThemeData(
        primarySwatch: Colors.blue,
      ),
      home: const MyHomePage(),
    );
  }
}

class MyHomePage extends StatelessWidget {
  const MyHomePage({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: AppConfig.isAdminApp ? _AdminBody(key: key) : _NonAdminBody(key: key),
    );
  }
}

class _AdminBody extends StatelessWidget {
  const _AdminBody({super.key});

  @override
  Widget build(BuildContext context) {
    return const Center(
      child: Text(
        'This is the admin UI.',
        style: TextStyle(fontSize: 22),
      ),
    );
  }
}

class _NonAdminBody extends StatelessWidget {
  const _NonAdminBody({super.key});

  @override
  Widget build(BuildContext context) {
    return const Center(
      child: Text(
        'This is the non-admin UI.',
        style: TextStyle(fontSize: 22),
      ),
    );
  }
}
```

As you can see, we have `_AdminBody()` class and `_NonAdminBody()` class which will help us render UIs depending on the app we are running.

On running both app flavors, we will have two different apps created with a single codebase.

<img src="https://dev-to-uploads.s3.amazonaws.com/uploads/articles/henef2uvke22feglk2tp.png" width=300 height="500"/>

## Conclusion

We learned how we can have two different apps created with different UIs using a single codebase. I hope this blog post will be helpful for some of you reading if you ever encounter a situation where you’d have to create a similar project.

If you wish to see some Flutter projects with proper architecture, follow me on [GitHub](https://github.com/Biplab-Dutta). I am also active on Twitter [@b_plab](https://twitter.com/b_plab98).

**[Source Code](https://github.com/Biplab-Dutta/product_flavor_demo)**

**My Socials:**

- [GitHub](https://github.com/Biplab-Dutta)

- [LinkedIn](https://www.linkedin.com/in/biplab-dutta-43774717a/)

- [Twitter](https://twitter.com/b_plab98)

Until next time, happy coding!!! 👨‍💻

— Biplab Dutta

## Credit

[Glitch](https://glitchyhitchy.medium.com/) for the preview image.
