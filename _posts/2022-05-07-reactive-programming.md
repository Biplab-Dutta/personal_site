---
title: A Taste of Reactive Programming in Flutter with RxDart and flutter_bloc
date: 2022-05-07 17:00:00 +545
categories: [flutter, reactive programming]
tags: [flutter, dart, reactive programming, rxdart, stream]
---

![Reactive Programming](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/kuk5hnesqsh0ef23mufu.jpg)
*Image Source: [West Agile Labs](https://www.westagilelabs.com/blog/five-interesting-facts-about-reactive-programming-frameworks/)*

When it comes to reactive programming, not many developers, especially junior developers feel comfortable with it. The base for reactive programming in Dart is **Stream**, a type in dart used to represent an asynchronous sequence of data. In this article, we will learn about what streams are, when to use them, and how to use them by building a simple flutter application and bringing everything into practice.

## Stream and StreamController
Before anything, let’s first talk about Streams as I already told you that it lays the foundation for reactive programming. Simply put, you can consider Stream as a flow of data that arrives asynchronously, meaning you don’t exactly know when a certain data is going to arrive. It is represented as `Stream<T>` where T is any data type.

Unlike Future, when you invoke a method returning Stream, the execution doesn’t get terminated once a result is **yield**ed (Stream’s equivalent to **return**). You will manually need to close the stream to terminate its operation.

When talking about Streams, you must also be familiar with **StreamController** . A StreamController is basically a manager for the stream. We use it to determine what type of stream we are interested in and for some other purposes.

To understand it more easily, imagine a pipe that has two ends. One for letting the water enter the pipe and the other for letting it out of the pipe. If you think about it, water freely flows inside of the pipe. Similarly, the StreamController has two ends — one for letting the data come in, and the other to let it exit. The data enters the StreamController via `Sink` . The data can also be modified before it is emitted.

![Stream Representation in Dart](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ia3q9suzkyr60lz2f7hn.png)
*Stream Representation (Image By: [Rajat Palankar](https://protocoderspoint.com/author/rajatrrpalankar/))*

Those parts of our application that are interested in our data stream need to listen to the emitted stream, also referred to as subscribing to the stream. Those who listen or subscribe to the stream are called **subscribers** or **listeners**. The stream can be listened to by one or many listeners. Depending on the number of listeners, the stream can be categorized into two parts:

- **Single-subscription Stream**: This type of stream only allows one listener during the whole lifetime of that stream. You can’t re-listen to the stream even after closing it.

- **Broadcast Stream**: This type of stream allows any number of listeners.


## Reactive Programming
Simply put, reactive programming is programming with asynchronous data streams.

> _If the repository in the data layer returns data in the form of Stream (which the ViewModel(s) will listen to), you can consider that reactive repository._
{: .prompt-info }

But how is it going to help us anyway? If this is what you have been thinking, let me demonstrate two images.

> _A picture is worth a thousand words — Henrik Ibsen_
{: .prompt-tip }

![Steps in performing Network Request](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ijfywt0dfdesaa2teb2w.png)
*Steps in performing Network Request — Future-based Approach (Image Source: [Android Developers](https://www.youtube.com/watch?v=fSB6_KE95bU&t=103s))*

We can clearly understand what the above image illustrates. It shows how a developer can obtain necessary data from the server by following the shown steps:

- First, the view is initialized.

- Then the ViewModel (or bloc) requests the necessary data. The communication takes place with the Repository.

- The Repository communicates with the relevant data sources for the required data.

- The data source performs some data fetching operations and returns the data back to the Repository.

- The Repository returns the data back to the ViewModel.

- And finally, the data is consumed by the UI.

While there is nothing wrong with this approach to performing network requests, in larger projects, it might be an overkill. If we add some data to our database, we would immediately have to call some methods like `loadData()` or `getUpdatedData()`.

> _`loadData()` is a one-time delivery of data meaning you shouldn’t be calling methods like `loadData()` time and again. The app must contain logic to ask for updates periodically._
{: .prompt-info }

What is the better way of doing the same thing then? And the answer to that would be having a **reactive repository**. To illustrate this, see this image.

![Steps in performing Network Request](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/kh2aora96rupg40br9u0.png)
*Steps in performing Network Request — Reactive Approach (Image Source: [Android Developers](https://www.youtube.com/watch?v=fSB6_KE95bU&t=129s))*

This diagram has fewer steps than what we saw earlier. But how does this work? Let’s have a look at it.

- First, the view is initialized and requests ViewModel for initial data and observes it. Observes is another term for listening or subscribing.

- Then the ViewModel requests Repository for the data and observes the Repository.

- And the Repository requests data from the data source and observes the data source.

Now, how will the data that will be coming from the data source be ultimately consumed by the UI if the data is not being returned back?

Now, this is where the Streams and Reactive approach shines. 🤩

Because the data source is being observed (or listened to) by the Repository, which also means that the data source will return a Stream of some data, the repository will immediately be notified when new data arrives.

Again, the Repository is being observed (or listened to) by the ViewModel, the ViewModel would immediately be notified when the Repository yields some stream of data.

And finally, as the ViewModel is observed by the UI, the UI gets proper data to display on the screen.

With this approach, we needn’t call methods like `loadData()` again and again as new data arrives. We only add incoming data to the Stream then the relevant listener(s) will automatically get notified about the changes. This is what we call a **Reactive Approach**.

Now, we will begin building a simple application, that will put all of our understanding regarding Streams and the Reactive approach into practice.

## Demo
We will build a very simple application that will initially display a list of names in a ListView. And we will add some more names but follow a reactive approach.

This is what our final application looks like.

![Flutter app](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/s5z8h379yargqik2guh9.gif)

## Creating flutter project
First, create a new Flutter project and get rid of the default counter app.
Also, add [Equatable](https://pub.dev/packages/equatable) and [RxDart](https://pub.dev/packages/rxdart) packages to your `pubspec.yaml` file.

```shell
flutter create stream_demo
```

I named the project `stream_demo` . You can name your project anything you like. Then inside your lib, create three folders and name them data, domain, and presentation. The `main.dart` file should also reside in the lib directory.

```
lib
 |__ data
 |__ domain
 |__ presentation
 |__ main.dart
```

## Development
### Domain Layer
We will begin with the **Domain Layer** first. Inside the domain folder, create a file `models.dart` . And paste the following code.
```dart
class User {
  final String name;

  User(this.name);
}

final allUsers = <User>[
  User('Biplab'),
  User('Aashish'),
  User('Aayam'),
  User('Saroj'),
  User('Ereema'),
  User('Manisha'),
];
```

### Data Layer
Next, we will work on the **Data Layer**. Create two files `repository.dart` and `remote_data_source.dart` inside the data folder.

Now, add the following code to `remote_data_source.dart` file.

```dart
import 'package:stream_demo/domain/models.dart';

class RemoteDataSource {
  const RemoteDataSource();

  /// This class, in a real project is supposed to make network requests to
  /// the server to get the data. But here we will just return our hard-coded
  /// list of users `List<User>`.
  Future<List<User>> getUsers() async {
    // make a network request
    await Future.delayed(const Duration(seconds: 2)); // mimicking network delay
    return allUsers;
  }
}
```
The above code simply returns a `List<User>` which will be returned by the Repository. So, add the following code to `repository.dart` file.

```dart
import 'dart:async';

import 'package:rxdart/rxdart.dart';
import 'package:stream_demo/data/some_data_source.dart';
import 'package:stream_demo/domain/models.dart';

class UserRepository {
  final RemoteDataSource _dataSource;

  UserRepository(
    RemoteDataSource dataSource,
  ) : _dataSource = dataSource {
    _dataSource.getUsers().then(
          (users) => _userStreamController.add(users),
        );
  }

  final _userStreamController = BehaviorSubject<List<User>>();

  Stream<List<User>> getUsers() => _userStreamController.stream;

  /* If there are multiple listeners of this stream, then
     edit the `getUsers()` method as:
    
      Stream<List<User>> getUsers() => _userStreamController.asBroadcastStream();
  */

  void addUser(User user) {
    final users = [..._userStreamController.value, user];

    /* Don't do:
     _userStreamController.value.add(user);   ❌
     As it will add the new user to the previous List instance (which
     is also called mutating the List) due to which
     the ListView in the UI will not update properly.
     So, we should return new List instance with updated values.
    
     Alternatively, we can also do:
     
     final users = List<User>.from(_userStreamController.value)..add(user);   ✅
     
     */

    _userStreamController.add(users);

    /* 
      And finally the data is added to the stream as soon as this method is 
      invoked, allowing other listeners of this stream know about the new
      update immediately.
    */
  }
}
```
Here, the `UserRepository` is dependent on the `RemoteDataSource`. In the constructor body, we call `getUsers()` and the list of users is added to the **StreamController**.

Also, we are using **BehaviorSubject** instead of a **StreamController**. BehaviorSubject comes from [RxDart](https://pub.dev/packages/rxdart) package. The package is simply an extension on Stream provided by Dart. Using BehaviorSubject, any new listeners when begin listening to the stream, they immediately get the lastly emitted Stream of data.

> _We can think that `BehaviorSubject` has some kind of a caching mechanism that caches the latest emitted Stream immediately._
{: .prompt-tip }

Also, notice how the `addUser()` method is implemented. We are simply adding the new user to a new `List` and adding the `List` to our `StreamController`. This way, whenever a new user is added, the listeners subscribed to this Stream (in the bloc) immediately get notified, and our UI updates immediately.

> _I have had one question in the past when I saw `Stream<List<User>>` being returned instead of `Stream<User>`. The main reasons for that are:_
>
> _When a new user arrives, we can’t tell if that should be at the top of our list or at the bottom, or somewhere in the middle._
>
> _And one major principle of the BLoC Pattern is to pass those data in the UI that will be represented in the UI and avoid having any additional business logic in the UI._
{: .prompt-info }

### Presentation Layer
Now, we can work on the **Presentation Layer**. It will consist of bloc, widgets, and pages. So, create three folders inside the presentation folder and name them bloc, widgets, and pages respectively.

We will begin with the bloc. Add three files — `user_bloc.dart`, `user_event.dart`, and `user_state.dart`.

```dart 
// user_event.dart

part of 'user_bloc.dart';

abstract class UserEvent extends Equatable {
  const UserEvent();

  @override
  List<Object> get props => [];
}

class UserRequested extends UserEvent {
  const UserRequested();
}

class UserAdded extends UserEvent {
  const UserAdded(this.user);

  final User user;

  @override
  List<Object> get props => [user];
}

class UserInputChanged extends UserEvent {
  const UserInputChanged(this.input);

  final String input;

  @override
  List<Object> get props => [input];
}
```

```dart
// user_state.dart

part of 'user_bloc.dart';

enum UserStatus { initial, loading, success, failure }

class UserState extends Equatable {
  const UserState({
    this.status = UserStatus.initial,
    this.users = const [],
    this.userInput = '',
  });

  final UserStatus status;
  final List<User> users;
  final String userInput;

  UserState copyWith({
    UserStatus? status,
    List<User>? users,
    String? userInput,
  }) =>
      UserState(
        status: status ?? this.status,
        users: users ?? this.users,
        userInput: userInput ?? this.userInput,
      );

  @override
  List<Object> get props => [status, users, userInput];
}
```


```dart
// user_bloc.dart

import 'package:bloc/bloc.dart';
import 'package:equatable/equatable.dart';
import 'package:stream_demo/data/repository.dart';
import 'package:stream_demo/domain/models.dart';

part 'user_event.dart';
part 'user_state.dart';

class UserBloc extends Bloc<UserEvent, UserState> {
  UserBloc({
    required UserRepository repository,
  })  : _repository = repository,
        super(const UserState()) {
    on<UserRequested>(_onUserRequested);
    on<UserAdded>(_onUserAdded);
    on<UserInputChanged>(_onUserInputChanged);
  }

  final UserRepository _repository;

  Future<void> _onUserRequested(
    UserRequested event,
    Emitter<UserState> emit,
  ) async {
    emit(state.copyWith(status: UserStatus.loading));

    /// We also could have used `stream.listen` however `emit.forEach()` is a
    /// newer and recommended approach when working with the flutter_bloc package.
    await emit.forEach<List<User>>(
      _repository.getUsers(),
      onData: (users) => state.copyWith(
        status: UserStatus.success,
        users: users,
      ),
      onError: (_, __) => state.copyWith(status: UserStatus.failure),
    );
  }

  void _onUserAdded(
    UserAdded event,
    Emitter<UserState> emit,
  ) {
    _repository.addUser(event.user);
    emit(state.copyWith(userInput: ''));
  }

  void _onUserInputChanged(
    UserInputChanged event,
    Emitter<UserState> emit,
  ) {
    emit(
      state.copyWith(
        userInput: event.input,
      ),
    );
  }
}
```

The event `UserRequested` is triggered as soon as the app is launched. The event `UserAdded` is triggered when we want to add the user to our existing list. And the `UserInputChanged` event exists to handle the event when we provide input to the text field.

Now to create UI, I'd suggest you check my [GitHub repo](https://github.com/Biplab-Dutta/flutter-stream-bloc-simplified) as the UI code is pretty simple and I don't want to create this article very long for the readers to read.

## Conclusion
This article aimed at making Streams and Reactive Programming easier to get used to. The reactive Approach can help in having a relatively small codebase, easier to manage, and most importantly having our application truly reactive.

If you have anything to tell me, please do so. I will be looking forward to all the feedback I receive. You can also follow me on Twitter at [@b_plab98](https://twitter.com/b_plab98) where I tweet about Flutter and Android.

## My Socials

- [GitHub](https://github.com/Biplab-Dutta/)

- [LinkedIn](https://www.linkedin.com/in/biplab-dutta-43774717a/)

- [Twitter](https://twitter.com/b_plab98)

Until next time, happy coding!!! 👨‍💻
— Biplab Dutta