# Flutter Login Tutorial

![advanced](https://img.shields.io/badge/level-advanced-red.svg)

> In the following tutorial, we're going to build a Firebase Login Flow in Flutter using the Bloc library.

![demo](./assets/gifs/flutter_firebase_login.gif)

## Setup

We'll start off by creating a brand new Flutter project

```bash
flutter create flutter_firebase_login
```

We can then replace the contents of `pubspec.yaml` with

```yaml
name: flutter_firebase_login
description: A new Flutter project.

version: 1.0.0+1

environment:
  sdk: ">=2.0.0-dev.68.0 <3.0.0"

dependencies:
  flutter:
    sdk: flutter
  cloud_firestore: ^0.9.7
  firebase_auth: ^0.8.1+4
  google_sign_in: ^4.0.1+1
  flutter_bloc: ^0.11.0
  equatable: ^0.2.0
  meta: ^1.1.6
  font_awesome_flutter: ^8.4.0

dev_dependencies:
  flutter_test:
    sdk: flutter

flutter:
  uses-material-design: true
  assets:
    - assets/
```

and then install all of our dependencies

```bash
flutter packages get
```

The last thing we need to do is follow the [firebase_auth usage instructions](https://pub.dartlang.org/packages/firebase_auth#usage) in order to hook up our application to firebase and enable [google_signin](https://pub.dartlang.org/packages/google_sign_in).

## User Repository

Just like in the [flutter login tutorial](./flutterlogintutorial.md), we're going to need to create our `UserRepository` which will be responsible for abstracting the underlying implementation for how we authenticate and retreive user information.

Let's create `user_repository.dart` and get started.

We can start by defining our `UserRepository` class and implementing the constructor. You can immediately see that the `UserRepository` will have a dependency on both `FirebaseAuth` and `GoogleSignIn`.

```dart
import 'package:firebase_auth/firebase_auth.dart';
import 'package:google_sign_in/google_sign_in.dart';

class UserRepository {
  final FirebaseAuth _firebaseAuth;
  final GoogleSignIn _googleSignIn;

  UserRepository({FirebaseAuth firebaseAuth, GoogleSignIn googleSignin})
      : _firebaseAuth = firebaseAuth ?? FirebaseAuth.instance,
        _googleSignIn = googleSignin ?? GoogleSignIn();
}
```

?> **Note:** If `FirebaseAuth` and/or `GoogleSignIn` are not injected into the `UserRepository`, then we instantiate them internally. This allows us to be able to inject mock instances so that we can easily test the `UserRepository`.

The first method we're going to implement we will call `signInWithGoogle` which will authenticate the user using the `GoogleSignIn` package.

```dart
Future<FirebaseUser> signInWithGoogle() async {
  final GoogleSignInAccount googleUser = await _googleSignIn.signIn();
  final GoogleSignInAuthentication googleAuth =
      await googleUser.authentication;
  final AuthCredential credential = GoogleAuthProvider.getCredential(
    accessToken: googleAuth.accessToken,
    idToken: googleAuth.idToken,
  );
  await _firebaseAuth.signInWithCredential(credential);
  return _firebaseAuth.currentUser();
}
```

Next, we'll implement a `signInWithCredentials` method which will allow users to sign in with their own credentials using `FirebaseAuth`.

```dart
Future<void> signInWithCredentials(String email, String password) {
  return _firebaseAuth.signInWithEmailAndPassword(
    email: email,
    password: password,
  );
}
```

Up next, we need to implement a `signUp` method which allows users to create an account if they choose not to use Google Sign In.

```dart
Future<void> signUp({String email, String password}) async {
  return await _firebaseAuth.createUserWithEmailAndPassword(
    email: email,
    password: password,
  );
}
```

We need to implement a `signOut` method so that we can give users the option to logout and clear their profile information from the device.

```dart
Future<void> signOut() async {
  return Future.wait([
    _firebaseAuth.signOut(),
    _googleSignIn.signOut(),
  ]);
}
```

Lastly, we will need two additional methods: `isSignedIn` and `getUser` to allow us to check if a user is already authenticated and to retrieve their information.

```dart
Future<bool> isSignedIn() async {
  final currentUser = await _firebaseAuth.currentUser();
  return currentUser != null;
}

Future<String> getUser() async {
  return (await _firebaseAuth.currentUser()).email;
}
```

?> **Note:** `getUser` is only returning the current user's email address for the sake of simplicity but we can define our own User model and populate it with a lot more information about the user in more complex applications.

Our finished `user_repository.dart` should look like this:

```dart
import 'dart:async';
import 'package:firebase_auth/firebase_auth.dart';
import 'package:google_sign_in/google_sign_in.dart';

class UserRepository {
  final FirebaseAuth _firebaseAuth;
  final GoogleSignIn _googleSignIn;

  UserRepository({FirebaseAuth firebaseAuth, GoogleSignIn googleSignin})
      : _firebaseAuth = firebaseAuth ?? FirebaseAuth.instance,
        _googleSignIn = googleSignin ?? GoogleSignIn();

  Future<FirebaseUser> signInWithGoogle() async {
    final GoogleSignInAccount googleUser = await _googleSignIn.signIn();
    final GoogleSignInAuthentication googleAuth =
        await googleUser.authentication;
    final AuthCredential credential = GoogleAuthProvider.getCredential(
      accessToken: googleAuth.accessToken,
      idToken: googleAuth.idToken,
    );
    await _firebaseAuth.signInWithCredential(credential);
    return _firebaseAuth.currentUser();
  }

  Future<void> signInWithCredentials(String email, String password) {
    return _firebaseAuth.signInWithEmailAndPassword(
      email: email,
      password: password,
    );
  }

  Future<void> signUp({String email, String password}) async {
    return await _firebaseAuth.createUserWithEmailAndPassword(
      email: email,
      password: password,
    );
  }

  Future<void> signOut() async {
    return Future.wait([
      _firebaseAuth.signOut(),
      _googleSignIn.signOut(),
    ]);
  }

  Future<bool> isSignedIn() async {
    final currentUser = await _firebaseAuth.currentUser();
    return currentUser != null;
  }

  Future<String> getUser() async {
    return (await _firebaseAuth.currentUser()).email;
  }
}
```

Next up, we're going to build our `AuthenticationBloc` which will be responsible for handling the `AuthenticationState` of the application in response to `AuthenticationEvents`.

## Authentication States

We need to determine how we’re going to manage the state of our application and create the necessary blocs (business logic components).

At a high level, we’re going to need to manage the user’s Authentication State. A user's authentication state can be one of the following:

- uninitialized - waiting to see if the user is authenticated or not on app start.
- authenticated - successfully authenticated
- unauthenticated - not authenticated

Each of these states will have an implication on what the user sees.

For example:

- if the authentication state was uninitialized, the user might be seeing a splash screen
- if the authentication state was authenticated, the user might see a home screen.
- if the authentication state was unauthenticated, the user might see a login form.

> It's critical to identify what the different states are going to be before diving into the implementation.

Now that we have our authentication states identified, we can implement our `AuthenticationState` class.

Create a folder/directory called `authentication_bloc` and we can create our authentication bloc files.

```sh
├── authentication_bloc
│   ├── authentication_bloc.dart
│   ├── authentication_event.dart
│   ├── authentication_state.dart
│   └── bloc.dart
```

?> **Tip:** You can use the [IntelliJ](https://plugins.jetbrains.com/plugin/12129-bloc-code-generator) or [VSCode](https://marketplace.visualstudio.com/items?itemName=FelixAngelov.bloc#overview) extensions to autogenerate the files for you.

```dart
import 'package:equatable/equatable.dart';
import 'package:meta/meta.dart';

@immutable
abstract class AuthenticationState extends Equatable {
  AuthenticationState([List props = const []]) : super(props);
}

class Uninitialized extends AuthenticationState {
  @override
  String toString() => 'Uninitialized';
}

class Authenticated extends AuthenticationState {
  final String displayName;

  Authenticated(this.displayName) : super([displayName]);

  @override
  String toString() => 'Authenticated { displayName: $displayName }';
}

class Unauthenticated extends AuthenticationState {
  @override
  String toString() => 'Unauthenticated';
}
```

?> **Note**: The [`equatable`](https://pub.dartlang.org/packages/equatable) package is used in order to be able to compare two instances of `AuthenticationState`. By default, `==` returns true only if the two objects are the same instance.

?> **Note**: `toString` is overridden to make it easier to read an `AuthenticationState` when printing it to the console or in `Transitions`.

!> Since we're using `Equatable` in order to allow us to compare different instances of `AuthenticationState` we need to pass any properties to the superclass. Without `super([displayName])`, we will not be able to properly compare different instances of `Authenticated`.

## Authentication Events

Now that we have our `AuthenticationState` defined we need to define the `AuthenticationEvents` which our `AuthenticationBloc` will be reacting to.

We will need:

- an `AppStarted` event to notify the bloc that it needs to check if the user is currently authenticated or not.
- a `LoggedIn` event to notify the bloc that the user has successfully logged in.
- a `LoggedOut` event to notify the bloc that the user has successfully logged out.

```dart
import 'package:equatable/equatable.dart';
import 'package:meta/meta.dart';

@immutable
abstract class AuthenticationEvent extends Equatable {
  AuthenticationEvent([List props = const []]) : super(props);
}

class AppStarted extends AuthenticationEvent {
  @override
  String toString() => 'AppStarted';
}

class LoggedIn extends AuthenticationEvent {
  @override
  String toString() => 'LoggedIn';
}

class LoggedOut extends AuthenticationEvent {
  @override
  String toString() => 'LoggedOut';
}
```

## Authentication Barrel File

Before we get to work on the `AuthenticationBloc` implementation, we will export all authentication bloc files from our `authentication_bloc/bloc.dart` barrel file. This will allow us import the `AuthenticationBloc`, `AuthenticationEvents`, and `AuthenticationState` with a single import later on.

```dart
export 'authentication_bloc.dart';
export 'authentication_event.dart';
export 'authentication_state.dart';
```

## Authentication Bloc

Now that we have our `AuthenticationState` and `AuthenticationEvents` defined, we can get to work on implementing the `AuthenticationBloc` which is going to manage checking and updating a user's `AuthenticationState` in response to `AuthenticationEvents`.

We'll start off by creating our `AuthenticationBloc` class.

```dart
import 'dart:async';
import 'package:bloc/bloc.dart';
import 'package:meta/meta.dart';
import 'package:flutter_firebase_login/authentication_bloc/bloc.dart';
import 'package:flutter_firebase_login/user_repository.dart';

class AuthenticationBloc
    extends Bloc<AuthenticationEvent, AuthenticationState> {
  final UserRepository _userRepository;

  AuthenticationBloc({@required UserRepository userRepository})
      : assert(userRepository != null),
        _userRepository = userRepository;
```

?> **Note**: Just from reading the class definition, we already know this bloc is going to be converting `AuthenticationEvents` into `AuthenticationStates`.

?> **Note**: Our `AuthenticationBloc` has a dependency on the `UserRepository`.

We can start by overriding `initialState` to the `AuthenticationUninitialized()` state.

```dart
@override
AuthenticationState get initialState => Uninitialized();
```

Now all that's left is to implement `mapEventToState`.

```dart
@override
Stream<AuthenticationState> mapEventToState(
  AuthenticationEvent event,
) async* {
  if (event is AppStarted) {
    yield* _mapAppStartedToState();
  } else if (event is LoggedIn) {
    yield* _mapLoggedInToState();
  } else if (event is LoggedOut) {
    yield* _mapLoggedOutToState();
  }
}

Stream<AuthenticationState> _mapAppStartedToState() async* {
  try {
    final isSignedIn = await _userRepository.isSignedIn();
    if (isSignedIn) {
      final name = await _userRepository.getUser();
      yield Authenticated(name);
    } else {
      yield Unauthenticated();
    }
  } catch (_) {
    yield Unauthenticated();
  }
}

Stream<AuthenticationState> _mapLoggedInToState() async* {
  yield Authenticated(await _userRepository.getUser());
}

Stream<AuthenticationState> _mapLoggedOutToState() async* {
  yield Unauthenticated();
  _userRepository.signOut();
}
```

We created separate private helper functions to convert each `AuthenticationEvent` into the propert `AuthenticationState` in order to keep `mapEventToState` clean and easy to read.

?> **Note:** We are using `yield*` (yield-each) in `mapEventToState` to separate the event handlers into their own functions. `yield*` inserts all the elements of the subsequence into the sequence currently being constructed, as if we had an individual yield for each element.

Our complete `authentication_bloc.dart` should now look like this:

```dart
import 'dart:async';
import 'package:bloc/bloc.dart';
import 'package:meta/meta.dart';
import 'package:flutter_firebase_login/authentication_bloc/bloc.dart';
import 'package:flutter_firebase_login/user_repository.dart';

class AuthenticationBloc
    extends Bloc<AuthenticationEvent, AuthenticationState> {
  final UserRepository _userRepository;

  AuthenticationBloc({@required UserRepository userRepository})
      : assert(userRepository != null),
        _userRepository = userRepository;

  @override
  AuthenticationState get initialState => Uninitialized();

  @override
  Stream<AuthenticationState> mapEventToState(
    AuthenticationEvent event,
  ) async* {
    if (event is AppStarted) {
      yield* _mapAppStartedToState();
    } else if (event is LoggedIn) {
      yield* _mapLoggedInToState();
    } else if (event is LoggedOut) {
      yield* _mapLoggedOutToState();
    }
  }

  Stream<AuthenticationState> _mapAppStartedToState() async* {
    try {
      final isSignedIn = await _userRepository.isSignedIn();
      if (isSignedIn) {
        final name = await _userRepository.getUser();
        yield Authenticated(name);
      } else {
        yield Unauthenticated();
      }
    } catch (_) {
      yield Unauthenticated();
    }
  }

  Stream<AuthenticationState> _mapLoggedInToState() async* {
    yield Authenticated(await _userRepository.getUser());
  }

  Stream<AuthenticationState> _mapLoggedOutToState() async* {
    yield Unauthenticated();
    _userRepository.signOut();
  }
}
```

Now that we have our `AuthenticationBloc` fully implemented, let’s get to work on the presentational layer.

## App

## Bloc Delegate

## Splash Screen

The first thing we’ll need is a `SplashScreen` widget which will be rendered while our `AuthenticationBloc` determines whether or not a user is logged in.

Let's create `splash_screen.dart` and implement it!

```dart
import 'package:flutter/material.dart';

class SplashScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Center(child: Text('Splash Screen')),
    );
  }
}
```

As you can tell, this widget is super minimal and you would probably want to add some sort of image or animation in order to make it look nicer. For the sake of simplicity, we're just going to leave it as is and move on to implementing the login flow.

## Login States

## Login Events

## Login Barrel File

## Login Bloc

## Validators

## Login Screen

## Login Button

## Google Login Button

## Create Account Button

## Login Form

## Home Screen

Next, we will need to create our `HomeScreen` so that we can navigate users there once they have successfully logged in. In this case, our `HomeScreen` will allow the user to logout and also will display their current name (email).

Let's create `home_screen.dart` and get started.

```dart
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:flutter_firebase_login/authentication_bloc/bloc.dart';

class HomeScreen extends StatelessWidget {
  final String name;

  HomeScreen({Key key, @required this.name}) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('Home'),
        actions: <Widget>[
          IconButton(
            icon: Icon(Icons.exit_to_app),
            onPressed: () {
              BlocProvider.of<AuthenticationBloc>(context).dispatch(
                LoggedOut(),
              );
            },
          )
        ],
      ),
      body: Column(
        mainAxisAlignment: MainAxisAlignment.spaceEvenly,
        children: <Widget>[
          Center(child: Text('Welcome $name!')),
        ],
      ),
    );
  }
}
```

`HomeScreen` is a `StatelessWidget` that requires a `name` to be injected so that it can render the welcome message. It also uses `BlocProvider` in order to access the `AuthenticationBloc` via `BuildCOntext` so that when a user pressed the logout button, we can dispatch the `LoggedOut` event.

## Register States

## Register Events

## Register Barrel File

## Register Bloc

## Register Screen

## Register Button

## Register Form
