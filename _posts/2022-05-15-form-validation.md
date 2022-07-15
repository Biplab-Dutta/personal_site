---
author: Biplab
title: Form Validation in Flutter using flutter_bloc and Functional Programming (Either)—A Proper Way
date: 2022-05-15 15:00:00 +545
categories: [flutter]
tags: [flutter, dart, functional programming, either, flutter_bloc]
image:
  path: https://raw.githubusercontent.com/Biplab-Dutta/personal_site/main/preview_images/form_validation.jpg
---

Dealing with forms is a very common task that we encounter as mobile application developers. With forms come form validation. It is necessary to show relevant warnings to the users when they don’t fill-up the form as they were supposed to. In order to do so, we need to write certain validation logic. However, the declarative UI approach in flutter results in many developers writing their validation logic right in the UI code which is **BAD, VERY BAD**.

Are you writing your validation logic in the UI? If yes, then this article is for you. I will be talking about a proper way how can deal with form validation that doesn’t just work but is architecturally clean and reasonable.

> _The approach that I will be sharing and which I often use in my personal projects is inspired by [Reso Coder](https://www.youtube.com/c/ResoCoder)’s tutorial on [Domain-driven design](https://www.youtube.com/playlist?list=PLB6lc7nQ1n4iS5p-IezFFgqP6YvAJy84U)._
{: .prompt-info }

## Form Validation in the UI (Bad Approach)
Let’s take a look at a snippet that validates a form with the validation logic in the UI.

```dart
class MyCustomForm extends StatefulWidget {
  const MyCustomForm({super.key});

  @override
  MyCustomFormState createState() {
    return MyCustomFormState();
  }
}

class MyCustomFormState extends State<MyCustomForm> {
  
  final _formKey = GlobalKey<FormState>();

  @override
  Widget build(BuildContext context) {
    return Form(
      key: _formKey,
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          TextFormField(
            validator: (value) {
              if (value == null || value.isEmpty) {   // Validation Logic
                return 'Please enter some text';
              }
              return null;
            },
          ),
          Padding(
            padding: const EdgeInsets.symmetric(vertical: 16.0),
            child: ElevatedButton(
              onPressed: () {
                if (_formKey.currentState!.validate()) {
                  ScaffoldMessenger.of(context).showSnackBar(
                    const SnackBar(content: Text('Processing Data')),
                  );
                }
              },
              child: const Text('Submit'),
            ),
          ),
        ],
      ),
    );
  }
}
```
As you can see, this is how someone would perform form validation with all validation logic right in the UI. It definitely works as intended but it is bad from an architectural point of view.

> _Many might argue that for something as simple as this is, it is not necessary to be concerned about architecture and do it as shown above but as developers, we should have developer ethics and we should do things in the right way. There should never be room for a simple workaround._
{: .prompt-info }

Now, if you are further reading this article, I assume you agree with me. Now let’s take a look at how to perform form validation — the right way.

Before anything, let’s add some dependencies and dev-dependencies that we will require for this project. You will be needing [dartz](https://pub.dev/packages/dartz), [equatable](https://pub.dev/packages/equatable), [flutter_bloc](https://pub.dev/packages/flutter_bloc), and [freezed_annotation](https://pub.dev/packages/freezed_annotation) as your dependencies. Also, [build_runner](https://pub.dev/packages/build_runner) and [freezed](https://pub.dev/packages/freezed) as your dev-dependencies.

## Domain Layer
Let’s have a look at what we often see in simple projects where a `login()` method is implemented.
```dart
Future<void> login({
   required String email,
   required String password,
}) async {}
```
This is generally the method signature for `login()` method. While we can definitely do something with the method shown above, it won’t prevent some developers in the future from messing up.
For example, one can call the method as
```dart
await login(
   email: "email123", 
   password: "pw",
);
```
Syntactically, it is fine. There’s nothing wrong with what we have done here. But logically, the email can’t just be `email123` and the password shouldn’t just be two characters long. So, how can we deal with issues as such, enforcing us to pass only email-like input to email and the same for password?

And the answer to that is creating `EmailAddress` and `Password` class. Having a class for each attribute will allow us to write some custom logic as well which we will see in a moment.
```dart
class EmailAddress {
  const EmailAddress(this.value);
  final String value;
}
```
Now, we have a `EmailAddress` class with a value property of String type. But this is no different from what I showed earlier because the property `value` is still a `String` and any string can be passed to it. If we think about the property `value` , then it can either be a legitimate String value or an illegitimate String. For example, if the property value for `EmailAddress` class is `‘abc@gmail.com’` then, this is a legitimate string value for email. But if the property `value` is something like `‘abc’` then, it is an illegitimate string value.

Now, what data type can we use for the property value so as to tell that it can have either a legitimate value or an illegitimate value? And the answer to that will be using **[Either](https://medium.com/disney-streaming/option-either-state-and-io-imperative-programming-in-a-functional-world-8e176049af81)** type from package [dartz](https://pub.dev/packages/dartz).

> _`Either` is an entity whose value can be of two different types, called left and right. By convention, `Right` is for the success case and `Left` is for the error one. It’s a common pattern in the functional community._
{: .prompt-tip }


```dart
class EmailAddress {
  const EmailAddress(this.value);
  final Either<ValueFailure, String> value;
}
```
Now, using `Either` type, we can tell the compiler that value can possibly have one out of two types. In this case, the property value can either be of a `ValueFailure` type or a `String` type.

Now, what is a `ValueFailure`? `ValueFailure` is simply a union to represent an invalid email string. We can create unions using [freezed](https://pub.dev/packages/freezed) package.

```dart
part 'value_failure.freezed.dart';

@freezed
class ValueFailure with _$ValueFailure {
  const factory ValueFailure.invalidEmail({
    required String failedValue,
  }) = _InvalidEmail;
}
```
Now, run the command

```shell
flutter pub run build_runner watch --delete-conflicting-outputs
```
This should generate a bunch of code and all the errors should be gone.

Now, if we take a look at our `EmailAddress` class, everything looks fine except for the validation logic. We somehow need to add the validation logic while we are in the domain layer. It would be great if we could run the validation as soon as `EmailAddress` class was instantiated.
Fortunately, we can do so with the help of a **factory constructor**. First, we will create a private constructor so that it can’t be used to create an instance of `EmailAddress` class. And then add a factory constructor which will act as a default constructor. The factory constructor will take in a string as an input which will undergo validation.

> _We shall be using Regex to perform email validation._
{: .prompt-tip }

```dart
class EmailAddress extends Equatable {
  factory EmailAddress(String input) =>
      EmailAddress._(_validateEmailAddress(input));

  const EmailAddress._(this.value);

  final Either<ValueFailure, String> value;

  @override
  List<Object?> get props => [value];
}

Either<ValueFailure, String> _validateEmailAddress(String input) {
  const emailRegex =
      r"""^[a-zA-Z0-9.a-zA-Z0-9.!#$%&'*+-/=?^_`{|}~]+@[a-zA-Z0-9]+\.[a-zA-Z]+""";
  if (RegExp(emailRegex).hasMatch(input)) {
    return right(input);
  } else {
    return left(
      ValueFailure.invalidEmail(failedValue: input),
    );
  }
}
```
We can see that if we try to do something like

```dart
var email = EmailAddress('abc@gmail.com');
```
then immediately, the passed in string will go through our validation logic and either return `ValueFailure` or `String`. Also, notice that we are extending our `EmailAddress` class with [Equatable](https://pub.dev/packages/equatable) to enforce value equality over reference equality which requires us to override the `props` getter.

And this is it.

Now, we do the same thing for the `Password` class too. The validation logic will only differ and the rest remains the same. Also, we will need to add another redirecting constructor in our `ValueFailure` union class. Therefore, our `value_failure.dart` and `password.dart` would look like this:
```dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'value_failure.freezed.dart';

@freezed
class ValueFailure with _$ValueFailure {
  const factory ValueFailure.invalidEmail({
    required String failedValue,
  }) = _InvalidEmail;

  const factory ValueFailure.shortPassword({
    required String failedValue,
  }) = _Password;
}
```

```dart
class Password extends Equatable {
  factory Password(String input) => Password._(_validatePassword(input));

  const Password._(this.value);

  final Either<ValueFailure, String> value;

  @override
  List<Object?> get props => [value];
}

Either<ValueFailure, String> _validatePassword(String input) {
  if (input.length >= 5) {
    return right(input);
  } else {
    return left(
      ValueFailure.shortPassword(failedValue: input),
    );
  }
}
```
## Presentation Layer
Now, we will begin working with our presentation layer. Let me show you the final result of our application.

![flutter form validation demo](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/796f898wzkv7fxxde3bz.gif)

We will begin with creating our bloc.
### Event class
Considering the events or actions that users can perform to interact with the UI, the events can be

- **EmailChanged** (when the user adds an input character to the email text field)

- **PasswordChanged** (when the user adds an input character to the password text field)

- **ObscurePasswordToggled** (when the user taps on the show password or hide password option in the password text field)

- **LoginSubmitted** (when the user taps on the login button)

We will be creating a union class to represent events in our project.

```dart
part of 'login_form_bloc.dart';

@freezed
class LoginFormEvent with _$LoginFormEvent {
  const factory LoginFormEvent.emailChanged(String emailString) = _EmailChanged;

  const factory LoginFormEvent.passwordChanged(String passwordString) =
      _PasswordChanged;

  const factory LoginFormEvent.obscurePasswordToggled() =
      _ObscurePasswordToggled;

  const factory LoginFormEvent.loginSubmitted() = _LoginSubmitted;
}
```
### State class
What field should be present in the state class? We need to pass back the validated email and password to our UI.
Also, on pressing the login button, a loading indicator appears until a response is received from the backend. So, we will need a `isSubmitting` `boolean` field that will be false by default.
We don’t want the validation logic to kick in as soon as our app starts. Only once the login button is pressed, and if the email and password combination is invalid then we want the warnings to begin getting displayed. So, we also need a `showErrorMessage` `boolean` field that will be false by default.
We also have a **_show/hide password_** option and the password will remain hidden (obscured) by default. So, we need an `obscurePassword` `boolean` field which will be true by default.
After tapping on login, we will either succeed or fail. In this demo, we will show a snack bar if we log in when having a valid email and password. In a real-world application, you would show a snack bar with a proper message if the user is unable to log in or navigate to the home screen if login is successful. Since we will either succeed or fail the login process, we will need an `authSuccessOrFailure` `Either<AuthFailure, Unit>?` field. It is a nullable field because as the app starts we can’t tell if the login process is successful or not. So, if the `authSuccessOrFailure` is null that means we haven’t tried logging in yet.
`Unit` is a data type that comes from [dartz](https://pub.dev/packages/dartz) package and is equivalent to `void`.

```dart
part of 'login_form_bloc.dart';

@freezed
class LoginFormState with _$LoginFormState {
  const factory LoginFormState({
    required EmailAddress emailAddress,
    required Password password,
    @Default(false) bool isSubmitting,
    @Default(false) bool showErrorMessage,
    @Default(true) bool obscurePassword,
    Either<AuthFailure, Unit>? authFailureOrSuccess,
    // Unit comes from Dartz package and is equivalent to void.
  }) = _LoginFormState;

  factory LoginFormState.initial() => LoginFormState(
        emailAddress: EmailAddress(''),
        password: Password(''),
      );
}
```
`AuthFailure` is also a union class to represent authentication failure and because the authentication process might fail because of several reasons, we are using a union class.
```dart
part 'auth_failure.freezed.dart';

@freezed
class AuthFailure with _$AuthFailure {
  const factory AuthFailure.invalidEmailAndPasswordCombination() =
      _InvalidEmailAndPasswordCombination;
  const factory AuthFailure.serverError() = _ServerError;
}
```

> _Since we ran the **build_runner watch** command earlier, we needn’t run the build command again. For some reason, if the build_runner watch command has stopped running, you will need to re-run the command._
{: .prompt-tip }

```shell
flutter pub run build_runner watch --delete-conflicting-outputs
```

### Bloc
This is where the presentation logic is located.

```dart
part 'login_form_bloc.freezed.dart';
part 'login_form_event.dart';
part 'login_form_state.dart';

class LoginFormBloc extends Bloc<LoginFormEvent, LoginFormState> {
  LoginFormBloc() : super(LoginFormState.initial()) {
    on<LoginFormEvent>(
      (event, emit) async {
        await event.when<FutureOr<void>>(
          emailChanged: (emailString) => _onEmailChanged(emit, emailString),
          passwordChanged: (passwordString) =>
              _onPasswordChanged(emit, passwordString),
          obscurePasswordToggled: () => _onObscurePasswordToggled(emit),
          loginSubmitted: () => _onLoginSubmitted(emit),
        );
      },
    );
  }

  void _onEmailChanged(Emitter<LoginFormState> emit, String emailString) {
    emit(
      state.copyWith(
        emailAddress: EmailAddress(emailString),
        authFailureOrSuccess: null,
      ),
    );
  }

  void _onPasswordChanged(Emitter<LoginFormState> emit, String passwordString) {
    emit(
      state.copyWith(
        password: Password(passwordString),
        authFailureOrSuccess: null,
      ),
    );
  }

  void _onObscurePasswordToggled(Emitter<LoginFormState> emit) {
    emit(state.copyWith(obscurePassword: !state.obscurePassword));
  }

  Future<void> _onLoginSubmitted(Emitter<LoginFormState> emit) async {
    final isEmailValid = state.emailAddress.value.isRight();
    final isPasswordValid = state.password.value.isRight();

    if (isEmailValid && isPasswordValid) {
      emit(
        state.copyWith(
          isSubmitting: true,
          authFailureOrSuccess: null,
        ),
      );

      // Perform network request to get a token.

      await Future.delayed(const Duration(seconds: 1));
    }
    emit(
      state.copyWith(
        isSubmitting: false,
        showErrorMessage: true,

        // Depending on the response received from the server after loggin in,
        // emit proper authFailureOrSuccess.

        // For now we will just see if the email and password were valid or not
        // and accordingly set authFailureOrSuccess' value.

        authFailureOrSuccess:
            (isEmailValid && isPasswordValid) ? right(unit) : null,
      ),
    );
  }
}
```
Now all that is left is to create two TextFields and wrap them with the BlocBuilder widget.

To check the remaining UI source code, I’d suggest you take a look at this [repo](https://github.com/Biplab-Dutta/form-validation) which contains the source code for the entire project.

## Conclusion
This article demonstrated how you can perform form validation in Flutter using proper techniques and without having any business logic in the UI. There are several other ways to do the same thing. One of them happens to be [formz](https://github.com/Biplab-Dutta/formz) package.

I hope after reading this article those who have been writing their validation logic in the UI, would now have such logic in the domain layer, keeping your presentation layer neat.

If you wish to see some Flutter projects with proper architecture, follow me on [GitHub](https://github.com/Biplab-Dutta). I am also active on Twitter [@b_plab](https://twitter.com/b_plab98).

**[Source Code](https://github.com/Biplab-Dutta/form-validation)**

**My Socials:**

- [GitHub](https://github.com/Biplab-Dutta)

- [LinkedIn](https://www.linkedin.com/in/biplab-dutta-43774717a/)

- [Twitter](https://twitter.com/b_plab98)

Until next time, happy coding!!! 👨‍💻

— Biplab Dutta