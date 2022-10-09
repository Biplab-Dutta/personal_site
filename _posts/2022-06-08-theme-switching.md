---
author: Biplab
title: Theme Switching & Persisting in Flutter using cubits and Stream
date: 2022-06-08 18:00:00 +545
categories: [flutter]
tags: [flutter, dart, flutter_bloc, stream, cubits, themes]
image:
  path: https://raw.githubusercontent.com/Biplab-Dutta/personal_site/main/preview_images/theme_switching.jpg
  alt: Preview Image Credit - raywenderlich.com
---

Every mobile app user prefers having an option to choose between multiple themes. Having decent themes available is also very crucial in enhancing the user experience. So, how can we do it effectively in Flutter? How can we have different configs set for each theme? This article ensures that you get a proper understanding of it and I’ll also talk a bit regarding `Stream` in Dart.

<!-- omit in toc -->
## Table of Contents

<!-- Chirpy theme by default provides a table of contents for desktop users. I am creating one explicitly for mobile users. -->

- [Demo](#demo)
- [Dependencies](#dependencies)
- [Let’s Code 👨‍💻](#lets-code-)
  - [Theme Configurations](#theme-configurations)
  - [Data Layer/Theme Repository](#data-layertheme-repository)
  - [Cubits/ViewModel/State Management](#cubitsviewmodelstate-management)
  - [main.dart](#maindart)
  - [App Widget](#app-widget)
  - [HomePage](#homepage)
- [Other solutions](#other-solutions)
- [Conclusion](#conclusion)
- [Credit](#credit)

## Demo

Let’s take a look at our final app.

![flutter-theme-switching-demo](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/bs1jwa9pgp858t2gyiy9.gif)

As we can see in the GIF, our app allows us to switch between dark theme and light theme. Also, the icon on the floating action button changes dynamically. And the chosen theme is persisted which can be witnessed in every app launch.

## Dependencies

Before we begin working on the code, let’s first include some external packages that we will need. Include [flutter_bloc](https://pub.dev/packages/flutter_bloc), [shared_preferences](https://pub.dev/packages/shared_preferences), and [equatable](https://pub.dev/packages/equatable) as your dependencies in `pubspec.yaml` file.

## Let’s Code 👨‍💻

Since this is a very simple app, I won’t be concerned about app architecture in this article. You can check my other articles if you want to learn about app architecture.

### Theme Configurations

First, start a new flutter project and get rid of the default counter app. Then inside the lib folder, create a file `app_theme.dart`.

```dart
import 'package:flutter/material.dart';

abstract class AppTheme {
  static ThemeData get lightTheme => ThemeData(
        scaffoldBackgroundColor: Colors.white,
        textTheme: ThemeData.light().textTheme.copyWith(
              bodyText1: const TextStyle(
                fontSize: 25,
                color: Colors.black,
              ),
              caption: const TextStyle(
                fontStyle: FontStyle.italic,
                fontSize: 15,
                color: Colors.black,
              ),
            ),
      );

  static ThemeData get darkTheme => ThemeData(
        scaffoldBackgroundColor: Colors.blueGrey.shade800,
        textTheme: ThemeData.dark().textTheme.copyWith(
              bodyText1: const TextStyle(
                fontSize: 25,
                color: Colors.white,
              ),
              caption: const TextStyle(
                fontStyle: FontStyle.italic,
                fontSize: 15,
                color: Colors.white,
              ),
            ),
      );
}
```

### Data Layer/Theme Repository

Then create a `theme_repository.dart` file inside the lib directory and paste the following code.

```dart
import 'package:shared_preferences/shared_preferences.dart';

abstract class ThemePersistence {
  Stream<CustomTheme> getTheme();
  Future<void> saveTheme(CustomTheme theme);
  void dispose();
}

enum CustomTheme { light, dark }

class ThemeRepository implements ThemePersistence {
  ThemeRepository({
    required SharedPreferences sharedPreferences,
  }) : _sharedPreferences = sharedPreferences;
  
  final SharedPreferences _sharedPreferences;

  @override
  Stream<CustomTheme> getTheme() {}

  @override
  Future<void> saveTheme(CustomTheme theme) {}

  @override
  void dispose() {}
}
```

As you can see, we created an abstract class `ThemePersistence`, and another class `ThemeRepository` that implements the abstract class. Also, we created an enum `CustomTheme` that has two values — `light` and `dark` because these are the themes that our app will have. The `ThemeRepository` class depends on the `SharedPreferences` instance that we will use for theme persistence.

Also, we can see that `getTheme()` returns a `Stream` and not a `Future`. The main reason behind using Stream is that listeners of this Stream can immediately be notified of the theme change and we needn’t call the `getTheme()` method again and again.

If we had a `Future` implementation for `getTheme()`, after every theme update, we’d have to invoke the `getTheme()` method which is not ideal.

> _getTheme() is a one-time delivery of data meaning that it should be called only once and any changes should be yielded in the form of a stream which the listeners will listen to._
{: .prompt-tip }

Now, we will add code to our `getTheme()` and `saveTheme()` methods.

```dart
import 'package:shared_preferences/shared_preferences.dart';

abstract class ThemePersistence {
  Stream<CustomTheme> getTheme();
  Future<void> saveTheme(CustomTheme theme);
  void dispose();
}

enum CustomTheme { light, dark }

class ThemeRepository implements ThemePersistence {
  ThemeRepository({
    required SharedPreferences sharedPreferences,
  }) : _sharedPreferences = sharedPreferences;

  final SharedPreferences _sharedPreferences;

  static const _kThemePersistenceKey = '__theme_persistence_key__';

  final _controller = StreamController<CustomTheme>();

  Future<void> _setValue(String key, String value) =>
      _sharedPreferences.setString(key, value);

  @override
  Stream<CustomTheme> getTheme() => _controller.stream;

  @override
  Future<void> saveTheme(CustomTheme theme) {
    _controller.add(theme);
    return _setValue(_kThemePersistenceKey, theme.name);
  }

  @override
  void dispose() => _controller.close();
}
```

We initialized a `StreamController` which will act as a manager for our `Stream<CustomTheme>`. The `saveTheme()` method is straightforward. First, it adds the theme that we want to save to the stream and then calls `_setValue()` which will persist the theme. The `_setValue()` method uses API from `SharedPreferences` to persist the chosen theme.

Everything looks fine. But if you see it carefully, the method `getTheme()` yields the stream from the controller. But initially, as the app is launched, there would be no stream in the controller. Then how do we deal with it?

The solution is simple. We can add a constructor body that would be executed as soon as the `ThemeRepository` class is instantiated. Also, `getTheme()` is a method that would be called in the very early stage. So, we need to make sure that before the `getTheme()` method is called, there is some stream value in the controller. So, add a `_init()` method in the constructor body and instantiate the `ThemeRepository` in the `main.dart` file later. Then, the final version of our `theme_repository.dart` file would look like this:

```dart
import 'dart:async';

import 'package:shared_preferences/shared_preferences.dart';

abstract class ThemePersistence {
  Stream<CustomTheme> getTheme();
  Future<void> saveTheme(CustomTheme theme);
  void dispose();
}

enum CustomTheme { light, dark }

class ThemeRepository implements ThemePersistence {
  ThemeRepository({
    required SharedPreferences sharedPreferences,
  }) : _sharedPreferences = sharedPreferences {
    _init();
  }

  final SharedPreferences _sharedPreferences;

  static const _kThemePersistenceKey = '__theme_persistence_key__';

  final _controller = StreamController<CustomTheme>();

  String? _getValue(String key) {
    try {
      return _sharedPreferences.getString(key);
    } catch (_) {
      return null;
    }
  }

  Future<void> _setValue(String key, String value) =>
      _sharedPreferences.setString(key, value);

  void _init() {
    final themeString = _getValue(_kThemePersistenceKey);
    if (themeString != null) {
      if (themeString == CustomTheme.light.name) {
        _controller.add(CustomTheme.light);
      } else {
        _controller.add(CustomTheme.dark);
      }
    } else {
      _controller.add(CustomTheme.light);
    }
  }

  @override
  Stream<CustomTheme> getTheme() async* {
    yield* _controller.stream;
  }

  @override
  Future<void> saveTheme(CustomTheme theme) {
    _controller.add(theme);
    return _setValue(_kThemePersistenceKey, theme.name);
  }

  @override
  void dispose() => _controller.close();
}
```

Also, you may have noticed I used a `CustomTheme` enum that I created in this file instead of using `ThemeMode` which is available in Flutter. The reason is simply to avoid including the material package in the data layer of our project as `ThemeMode` comes from the material package. In my opinion, the components of the material package are associated with the presentation layer and not the data layer.

### Cubits/ViewModel/State Management

Now, we will create a `ThemeCubit` and `ThemeClass` class that will be responsible for our state handling. Create a folder in the lib directory and name it `them_cubit`. Inside `theme_cubit`, create two dart files — `theme_cubit.dart` and `theme_state.dart`.

```dart
// theme_state.dart

part of 'theme_cubit.dart';

class ThemeState extends Equatable {
  const ThemeState({this.themeMode = ThemeMode.light}); // Default theme = light theme

  final ThemeMode themeMode;

  // `copyWith()` method allows us to emit brand new instance of ThemeState
  ThemeState copyWith({ThemeMode? themeMode}) => ThemeState(
        themeMode: themeMode ?? this.themeMode,
      );

  @override
  List<Object?> get props => [themeMode];
}
```

```dart
// theme_cubit.dart

import 'dart:async';

import 'package:equatable/equatable.dart';
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:theme_switching_demo/theme_repository.dart';

part 'theme_state.dart';

class ThemeCubit extends Cubit<ThemeState> {
  ThemeCubit({
    required ThemePersistence themeRepository,
  })  : _themeRepository = themeRepository,
        super(const ThemeState());

  final ThemePersistence _themeRepository;
  late StreamSubscription<CustomTheme> _themeSubscription;
  static late bool _isDarkTheme;  // used to determine if the current theme is dark

  void getCurrentTheme() {
    // Since `getTheme()` returns a stream, we listen to the output
    _themeSubscription = _themeRepository.getTheme().listen(
      (customTheme) {
        if (customTheme.name == CustomTheme.light.name) {
          // Since, `customTheme` is light, we set `_isDarkTheme` to false
          _isDarkTheme = false;
          emit(state.copyWith(themeMode: ThemeMode.light));
        } else {
          // Since, `customTheme` is dark, we set `_isDarkTheme` to true
          _isDarkTheme = true;
          emit(state.copyWith(themeMode: ThemeMode.dark));
        }
      },
    );
  }

  void switchTheme() {
    if (_isDarkTheme) {
      // Since, currentTheme is dark, after switching we want light theme to
      // be persisted.
      _themeRepository.saveTheme(CustomTheme.light);
    } else {
      // Since, currentTheme is light, after switching we want dark theme to
      // be persisted.
      _themeRepository.saveTheme(CustomTheme.dark);
    }
  }

  @override
  Future<void> close() {
    _themeSubscription.cancel();
    _themeRepository.dispose();
    return super.close();
  }
}
```

The code is self-explanatory and I have added all the necessary explanations through comments in the above code. Be sure to go through them.

### main.dart

Let’s add some code to the `main.dart` file.

```dart
import 'package:flutter/material.dart';
import 'package:shared_preferences/shared_preferences.dart';
import 'package:theme_switching_demo/app.dart';
import 'package:theme_switching_demo/theme_repository.dart';

Future<void> main() async {
  // required when using any plugin. In our case, it's shared_preferences
  WidgetsFlutterBinding.ensureInitialized();
  
  // Creating an instance of ThemeRepository that will invoke the `_init()` method
  // and populate the stream controller in the repository.
  final themeRepository = ThemeRepository(
    sharedPreferences: await SharedPreferences.getInstance(),
  );
  
  runApp(App(themeRepository: themeRepository));
}
```

### App Widget

Next, create a `app.dart` file in the lib directory and paste the following code.

```dart
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:theme_switching_demo/home_page.dart';
import 'package:theme_switching_demo/theme_cubit/theme_cubit.dart';
import 'package:theme_switching_demo/theme_repository.dart';
import 'package:theme_switching_demo/themes.dart';

class App extends StatelessWidget {
  const App({required this.themeRepository, super.key});

  final ThemeRepository themeRepository;

  @override
  Widget build(BuildContext context) {
    return RepositoryProvider.value(
      value: themeRepository,
      child: BlocProvider(
        create: (context) => ThemeCubit(
          themeRepository: context.read<ThemeRepository>(),
        )..getCurrentTheme(),
        child: const AppView(),
      ),
    );
  }
}

class AppView extends StatelessWidget {
  const AppView({super.key});

  @override
  Widget build(BuildContext context) {
    return BlocBuilder<ThemeCubit, ThemeState>(
      builder: (context, state) {
        return MaterialApp(
          debugShowCheckedModeBanner: false,
          title: 'Flutter Demo',
          theme: AppTheme.lightTheme, // If ThemeMode is ThemeMode.light, this is selected as app's theme
          darkTheme: AppTheme.darkTheme, // If ThemeMode is ThemeMode.dark, this is selected as app's theme
          
          // The themeMode is the most important property in showing
          // proper theme. The value comes from ThemeState class.
          themeMode: state.themeMode,
          home: const HomePage(),
        );
      },
    );
  }
}
```

### HomePage

Create a file `home_page.dart` and add the following code.

```dart
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:theme_switching_demo/theme_cubit/theme_cubit.dart';

class HomePage extends StatelessWidget {
  const HomePage({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Theme Switching Demo'),
      ),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Text(
              'Follow me on my socials',
              style: Theme.of(context).textTheme.bodyText1,
              // Depending on the current theme, the text is also rendered properly
              // If the theme is dark, text is white in color else black
            ),
            const SizedBox(height: 10),
            Text(
              'https://github.com/Biplab-Dutta',
              style: Theme.of(context).textTheme.caption,
            ),
            const SizedBox(height: 10),
            Text(
              'https://twitter.com/b_plab98',
              style: Theme.of(context).textTheme.caption,
            ),
          ],
        ),
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () => context.read<ThemeCubit>().switchTheme(),
        tooltip: 'Switch Theme',
        child: context.read<ThemeCubit>().state.themeMode == ThemeMode.light
            ? const Icon(Icons.dark_mode)
            : const Icon(Icons.light_mode),
      ),
    );
  }
}
```

## Other solutions

There are many other ways of doing the same thing that I showed in this article. You could also use [hydrated_bloc](https://pub.dev/packages/hydrated_bloc) instead of cubits to manage the state and persist it. However, I wanted to show you how you could have done the persisting if you were working with other state management solutions other than flutter_bloc.

## Conclusion

This article showed how we can include theme switching and persisting feature in our Flutter app using the best practices. I hope you all got to learn from my article and if there’s any feedback for me, drop a comment. I’ll be sure to upload another article in a few days again.

If you wish to see some Flutter projects with proper architecture, follow me on [GitHub](https://github.com/Biplab-Dutta). I am also active on Twitter [@b_plab](https://twitter.com/b_plab98).

**[Source Code for the project in this article](https://github.com/Biplab-Dutta/flutter_theme_switcher)**

**My Socials:**

- [GitHub](https://github.com/Biplab-Dutta)

- [LinkedIn](https://www.linkedin.com/in/biplab-dutta-43774717a/)

- [Twitter](https://twitter.com/b_plab98)

Until next time, happy coding!!! 👨‍💻

— Biplab Dutta

## Credit

[raywenderlich.com](https://www.raywenderlich.com/16628777-theming-a-flutter-app-getting-started) for the preview image.
