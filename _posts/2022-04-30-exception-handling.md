---
author: Biplab
title: Exception Handling with Functional Programming in Flutter (Either type)
date: 2022-04-30 14:00:00 +0545
categories: [flutter, functional programming]
tags: [flutter, dart, functional programming, either]
---

> _If you wish to read this article in Bahasa Indonesia, you can find it [here](https://t.co/rpfOpMm3o0). [Yunus Afghoni](https://twitter.com/ghonijee?s=20&t=lMK4d9e4jKn7jVCN6_7iLQ) has done good work taking this article as a reference and translating it to Bahasa Indonesia, with some subtle changes._
{: .prompt-info }

Every application needs some data sources to receive the data and display it in the UI. So, it becomes very crucial how we, as developers, perform network requests. Handling API responses in an effective manner also determine the success or failure of our application.

In this post, we will see how to perform such network requests effectively using [dio](https://pub.dev/packages/dio) and a bit of functional programming using the [dartz](https://pub.dev/packages/dartz) package. This will allow our overall architecture to remain consistent while making our project scalable and maintainable.

To follow along, make sure to include [dio](https://pub.dev/packages/dio) and [dartz](https://pub.dev/packages/dartz) as your dependencies in the `pubspec.yaml` file.

### Dio API Calls

First thing first, let’s take a look at how I used to make network requests in Flutter a while back 😂 when I was still a beginner.
```dart
class FeedRemoteDataSource {
  FeedRemoteDataSource(Dio dio) : _dio = dio;

  final Dio _dio;

  Future<List<Feed>> fetchFeeds() async {
    try {
      final response = await _dio.get('some url path');
      // Some other work
    } on DioError {
      // error handling
      // return empty list.
    }
  }
}
```
This was how I did my network request returning a List of something no matter what. It worked fine for small projects; however, it didn’t take much longer for me to realize that I had been doing it wrong. As you can see in the code snippet above, I was always returning a `List`, irrespective of the status code.

In real-world projects, we are likely to work in a multi-layered architecture which means our entire project could be divided into multiple layers.
The layers can be:

- **Presentation Layer** (consisting of UI code and ViewModel)

- **Domain Layer** (consisting of entities and use-cases (optional))

- **Data Layer** (consisting of [DTOs](https://en.wikipedia.org/wiki/Data_transfer_object), a repository, and data sources)

In this article, we will work closely with the Data Layer as it is what this article is about in the first place.

Now, getting back to the previous snippet. The `fetchFeeds()` method always returned a `List`. The presentation layer will then have no idea if the returned `List` was because of a successful request or unsuccessful. Because a successful request can also sometimes give an empty list if the database has no data. So, it was clear that I was doing it wrong ❌.
After spending some time looking at other people’s codes and tutorials 📖, I learned the importance of software architecture. 🤯

So, keeping proper architecture in our mind, let us redo the same request but in a ***proper*** way. 😎
Since I talked about architecture, let me begin with what our architecture for the data layer should look like.

> _Before we proceed any further, let me tell you that this is not the best architecture or the only architecture you can go with. Different people/teams have their own ideas regarding architecture and their own ways of doing things. However, every architecture makes sure to separate UI and business logic which is why software architectures exist._
{: .prompt-info }

### Data Layer Architecture

![Data Layer Architecture](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/bvp87zh3qnjv4a9pn3vo.png)
*Data Layer (Image by [Reso Coder](https://twitter.com/resocoder))*

So, our data layer would consist of three parts:

- **Data Transfer Objects (DTOs)**: They are the data layer representation of domain layer entities. They are also responsible for performing various data-related logic like conversion between Dart Object and JSON format and vice-versa. It also contains logic to convert domain layer entities into DTOs and vice-versa.

- **Data Sources**: It is where we perform all of our network requests and the result is passed to the repository. All network-related exceptions that may arise are also thrown here which are then caught by the repository and converted to `Failure` (about which we will see in a moment).

- **Repository**: The repository is what the view model communicates with. A repository also acts as a single gateway for data coming from several data sources. A repository is thus, essential in maintaining a single source of truth.

### Data Transfer Objects (DTOs)
First, let’s see how our `feed_dto.dart` would look like.
```dart
part 'user_dto.freezed.dart';
part 'user_dto.g.dart';

@freezed    // Import package freezed. [https://pub.dev/packages/freezed]
class FeedDTO with _$FeedDTO {
  const factory FeedDTO({
    required String title,
  }) = _FeedDTO;

  const FeedDTO._();

  factory FeedDTO.fromJson(Map<String, dynamic> json) =>
      _$FeedDTOFromJson(json);

  factory FeedDTO.fromDomain(Feed feed) => FeedDTO(
        title: feed.title,
      );

  Feed toDomain() => Feed(title: title);
}
```
You will also need to have [freezed](https://pub.dev/packages/freezed) included in your project as a dev-dependency along with [build_runner](https://pub.dev/packages/build_runner). Also, include [freezed_annotation](https://pub.dev/packages/freezed_annotation) as a dependency.

> _freezed is a code generator for data-classes/unions/pattern-matching/cloning._
{: .prompt-info }

Next, you will need to have the required code generated which [freezed](https://pub.dev/packages/freezed) and [build_runner](https://pub.dev/packages/build_runner) will take care of. Run the following command to initialize code generation.
```bash
flutter pub run build_runner watch --delete-conflicting-outputs
```

### Data Sources
In this section, we will perform our network request using the [dio](https://pub.dev/packages/dio) package. Any exceptions that need to be thrown will be thrown in this section which then is handled by the **Repository**.
```dart
class FeedRemoteDataSource {
  FeedRemoteDataSource(
    Dio dio,
  ) : _dio = dio;

  final Dio _dio;

  Future<List<FeedDTO>> getFeeds() async {
    try {
      final response = await _dio.get<String>('some url path');
      if (response.statusCode == 200) {
        // decode json response
        // return List<FeedDTO>
      } else {
        throw RestApiException(response.statusCode);   // Custom class implementing Exception whose constructor accepts int
      }
    } on DioError catch (e) {
      if (e.isNoConnectionError) {
        // handle no connection error
      } else if (e.response != null) {
        throw RestApiException(e.response?.statusCode);
      } else {
        rethrow;
      }
    }
  }
}

extension DioErrorX on DioError {
  bool get isNoConnectionError =>
      type == DioErrorType.other && error is SocketException;   // import 'dart:io' for SocketException
}
```

### Repository
This is the main gateway for the data coming from several data sources. Also, the **ViewModel** communicates with the **Repository** to get the data and display it in the UI. And the conversion between DTO and the domain-level entity is also performed here.

Now, how is our repository going to make it easy for us to handle exceptions so as to have a maintainable architecture? It’s simple. We use **[Either](https://medium.com/disney-streaming/option-either-state-and-io-imperative-programming-in-a-functional-world-8e176049af81)**.

> _`Either` is an entity whose value can be of two different types, called left and right. By convention, `Right` is for the success case and `Left` is for the error one. It’s a common pattern in the functional community._
{: .prompt-info }

It might be difficult to get a grasp on `Either` just by looking at its definition. So, let’s take a look at our repository implementation which will help us understand `Either` easily.

The `FeedRepositoryImpl` class implements `FeedRepository` which is a simple abstract class. The `FeedRepositoryImpl` is dependent on our `FetchRemoteDataSource`.
```dart
abstract class FeedRepository {
  Future<Either<Failure, List<Feed>>> getFeeds();
}

class FeedRepositoryImpl implements FeedRepository {
  FeedRepositoryImpl({
    required this.remoteDataSource,
  });

  final FeedRemoteDataSource remoteDataSource;

  @override
  Future<Either<Failure, List<Feed>>> getFeeds() async {
    try {
      final result = await remoteDataSource.getFeeds();
      return right(result.toDomainList);
    } on RestApiException {
      return left(
        const Failure.serverError(),
      );
    }
  }
}

extension DTOListToDomainList on List<FeedDTO> {
  List<Feed> get toDomainList => map((e) => e.toDomain()).toList();
}
```

Now, now, now!! What is that method returning `Future<Either<Failure, List<Feed>>>` ??? 🤯😵‍💫 And how can a method return two different data types? 😵‍💫

If you are thinking the same, then it’s just as simple as it can get.

`Either<A, B>` means that a method will return either `A` or `B` depending on the situation. It won’t return both `A` and `B`.

In our case, the `getFeeds()` returns `Future<Either<Failure, List<Feed>>>` meaning either `Failure` or `List<Feed>` . And because we are dealing with asynchronous code, we also have `Future`.

`Failure` is just a simple union class that is created using [freezed](https://pub.dev/packages/freezed) package.

```dart
part 'failure.freezed.dart';

@freezed
class Failure with _$Failure {
  const factory Failure.serverError() = _ServerError;
  const factory Failure.anotherFailure() = _AnotherFailure;
}
```

If you had run the `build_runner watch` command earlier, saving the above `failure.dart` file will automatically generate a bunch of code for you. Else run the command again.

```bash
flutter pub run build_runner watch --delete-conflicting-outputs
```

So, our repository implementation is pretty straightforward now. If the remote data source returns relevant data, then the repository will return `Right` i.e. `List<Feed>` else it returns `Left` i.e. `Failure`. Notice how the exceptions that were wildly being thrown from the remote data source are now gone because the repository implementation returns the Dart object. 🤩

This way, we also reduce the risk of the [error bubble](https://www.linkedin.com/pulse/error-handling-let-bubble-up-mihael-schmidt/).

### BLoC / ViewModel
So, how exactly are we going to deal with the obtained result from the **repository** in the presentation layer? For that, we will need to create a **bloc** that will be dependent on the **repository**. I prefer using [flutter_bloc](https://pub.dev/packages/flutter_bloc) for state management purposes.

> _You needn’t use flutter_bloc to follow along. Any state management solution is fine._ 👍️
{: .prompt-tip }

```dart
part of 'timeline_bloc.dart';

@freezed
class TimelineEvent with _$TimelineEvent {
  const factory TimelineEvent.feedFetched() = _FeedFetched;
  // add some other events too as desired...
}
```
```dart
part of 'timeline_bloc.dart';

@freezed
class TimelineState with _$TimelineState {
  const factory TimelineState.loading() = _Loading;
  const factory TimelineState.loaded({required List<Feed> feeds}) = _Loaded;
  const factory TimelineState.failed() = _Failed;
}
```
```dart
class TimelineBloc extends Bloc<TimelineEvent, TimelineState> {
  TimelineBloc(
    FeedRepository repository,
  )   : _repository = repository,
        super(const TimelineState.loading()) {
    on<TimelineEvent>(
      (event, emit) async {
        await event.when<Future<void>>(
          feedFetched: () => _onFeedFetched(emit),
        );
      },
    );
  }

  final FeedRepository _repository;

  Future<void> _onFeedFetched(Emitter<TimelineState> emit) async {
    final fetchedFeed = await _repository.getFeeds();
    fetchedFeed.fold<void>(     // fetchedFeed is a Either type. We use fold to say what to do for each case i.e. for `Failure` and `Success` cases.
      (failure) => emit(const TimelineState.failed()),
      (feeds) => emit(TimelineState.loaded(feeds: feeds)),
    );
  }
}
```
In the `_onFeedFetched()` method above, the instance of the class implementing **FeedRepository** (the abstract class) is used to call the `getFeeds()` method which returns an `Either` type (stored in `fetchedFeed`). We use `fold` method to emit proper state depending on the result of `_repository.getFeeds().`
The `fold` accepts to functions as its argument. The first function is used to perform an action when `Failure` is returned, whereas the second function is used to perform an action when a `Success` is returned. `Success` in our case refers to `List<Feed>`.

Now, from our UI, we can use [BlocBuilder](https://pub.dev/packages/flutter_bloc#blocbuilder) to rebuild our widget on certain state changes.

### Other Solutions
There are many other ways to effectively handle exceptions in our Flutter project. We can also rely on sealed classes. A good example of it can be found [here](https://getstream.io/blog/modeling-retrofit-responses/).

### Conclusion
In this article, you saw how to implement network requests in Flutter in a proper manner. We learned how we can use [dio](https://pub.dev/packages/dio), [freezed](https://pub.dev/packages/freezed), [dartz](https://pub.dev/packages/dartz), and a few other architectural overviews that can help us in making our app more maintainable, and testable and ultimately help us in becoming a better developer.

Also, this was my very first blog on Dev. I know there is room for improvement and therefore, I seek feedback from the community.

If you wish to see some Flutter projects with proper architecture, follow me on [GitHub](https://github.com/Biplab-Dutta/). I am also active on Twitter [@b_plab](https://twitter.com/b_plab98).

### My Socials

- [GitHub](https://github.com/Biplab-Dutta/)

- [LinkedIn](https://www.linkedin.com/in/biplab-dutta-43774717a/)

- [Twitter](https://twitter.com/b_plab98)

Until next time, happy coding!!! 👨‍💻

— Biplab Dutta