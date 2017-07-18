

Purpose
---

The purpose of this tutorial series is to become familiar with how Aqueduct works. That means this tutorial will take a less efficient approach than a normal work flow so that the fundamentals are covered. For example, you will create a new project manually, but in a real work flow you would use the `aqueduct` command line tool.

This tutorial will use Atom for editing code. It has the lowest barrier to entry and is easy to install. For heavy duty work, we recommend IntelliJ IDEA Community Edition with the Dart plugin. The IntelliJ Dart plugin is an official plugin.

Installing Dart
---

If you have Homebrew installed, run these commands from terminal:

```bash
brew tap dart-lang/dart
brew install dart
```

If you don't have Homebrew installed or you are on another platform, visit [https://www.dartlang.org/install](https://www.dartlang.org/install). It'll be quick, promise.

You should install Atom for editing your Dart code. You can get it from [https://atom.io](https://atom.io). Once Atom is installed, install the 'dartlang' package from 'dart-atom' (not any of the other ones that have a similar name).  

Creating a Project
---

Create a new directory named `quiz` (ensure that it is lowercase). In this directory, create a new file named `pubspec.yaml`. Dart uses this file to define your project and its dependencies (like iOS' `Info.plist` or Android's `AndroidManifest.xml`).

In the pubspec, enter the following markup:

```
name: quiz
description: A quiz web server
version: 0.0.1
author: Thor Odinson <thor@asgard.yg>

environment:
  sdk: '>=1.20.0 <2.0.0'

dependencies:
  aqueduct: any  
```

This pubspec declares an application named `quiz` (all Dart files, directories and application identifiers are snake_case), indicates that it can use a version of the Dart SDK between 1.20 and 2.0, and depends on the `aqueduct` package.

Next, you will fetch the dependencies of the `quiz` project - this will fetch the source for `aqueduct` from Dart's hosted package manager. If you are using Atom, you'll get a popup that tells you to do this and you can just click the button. (You may also right-click on file in Atom and select 'Pub Get'). If you aren't using Atom, from the command line, run the following from inside the `quiz` directory:

```bash
pub get
```

Dependencies get stored in the global cache directory, `~/.pub-cache`. Dependencies are referenced by files in your project's directory. These files are automatically generated by the previous command. You won't have to worry about that, though, since you'll never have to deal with it directly. Sometimes, it's just nice to know where things are. (There is one other file, called `pubspec.lock` that you do care about, but we'll chat about it later.)

With this dependency installed, your project can use Aqueduct. For this simple getting started guide, we won't structure a full project and just focus on getting an Aqueduct web server up and running. Create a new directory named `lib` and add a file to it named `quiz.dart`. The project should now look like this on the filesystem:

```
quiz/
  pubspec.yaml
  pubspec.lock
  lib/
    quiz.dart
```

At the top of this file, import the Aqueduct package and the `async` standard library:

```dart
import 'dart:async';

import 'package:aqueduct/aqueduct.dart';
```

Handling Requests
---

The structure of Aqueduct is like most server-side frameworks: a new request comes in and gets routed to code that will respond to it. At its core, Aqueduct request handling is made up of three types of objects: `Request`s, `Response`s and `RequestController`s. When an Aqueduct application receives an HTTP request, it creates a `Request` object. For every `Request`, a `Response` must be created and sent back. `RequestController`s handle the logic of taking a `Request`s and creating a `Response`s.

Examples of `RequestController` subclasses are `Router` and `HTTPController`. A router figures out the right `RequestController` to handle a request by inspecting the request's path. An `HTTPController` takes a request - after it's been through a `Router` - and calls one its methods depending on the request's HTTP method (e.g., GET, POST). That method returns a `Response` and the request is completed.

The `quiz` application will have a `Router` that will send requests with the path `/questions` to an instance of `QuestionController`. `QuestionController` is an `HTTPController` subclass that you will write - it will respond with a list of JSON questions. In `quiz.dart`, create this new type:

```dart
class QuestionController extends HTTPController {
  var questions = [
    "How much wood can a woodchuck chuck?",
    "What's the tallest mountain in the world?"
  ];

  @httpGet
  Future<Response> getAllQuestions() async {
    return new Response.ok(questions);
  }
}
```

The `QuestionController` class has a list of strings named `questions` and a method called `getAllQuestions`. The metadata above the method - `httpGet` - is important. This metadata tells the `QuestionController` to invoke `getAllQuestions` when it receives a GET request. Likewise, if this metadata were `httpPut` or `httpPost`, this controller would invoke this method for PUT or POST requests.

Methods declared in `HTTPController` subclasses with this metadata are called *responder methods*, because they respond to HTTP requests. This is the primary job of an `HTTPController` - to map requests to a responder method based on their HTTP method. A responder method must return an instance of `Future<Response>`. There are convenience constructors for common response status codes. In this example, `Response.ok` creates a `Response` with status code 200. The first argument to `Response.ok` is an object that will be encoded as the HTTP response body.

Now, we must create a `Router`. A `Router` will receive all requests in an application. During initialization, a router is given *routes* - patterns that look for matches in the path of an HTTP request - and `RequestController`s registered to receive requests for those routes. In your application, the `Router` will have instances of `QuestionController` registered for the route `/questions`. All initialization for Aqueduct applications happens in a `RequestSink`.

An application subclasses `RequestSink` and overrides a few of methods to handle initializing the application. The one required override is `setupRouter`. This method, as the name suggests, is where you set up the application's routes and `RequestController`s. The tools that run an Aqueduct application know how to find and create your `RequestSink` subclass in your application.

At the bottom of `quiz.dart`, create a `RequestSink` subclass:

```dart
class QuizRequestSink extends RequestSink {
  QuizRequestSink(ApplicationConfiguration options) : super (options);

  @override
  void setupRouter(Router router) {
    router
      .route("/questions")
      .generate(() => new QuestionController());
  }
}
```

A `RequestSink` subclass must have a constructor that takes an `ApplicationConfiguration` instance and forward it on to its superclass' constructor. These values are provided by a configuration file and are typically used to configure a `RequestSink`'s properties - like a database connection. Since the `quiz` app doesn't do much right now, we simply forward the configuration options on to the `super`'s constructor as required.

Routes are registered through the `route` method. When a router matches the path of an HTTP request to one of its registered routes, the `Request` is sent to the next `RequestController` for that route. If no registered route matches the path of the request, the `Router` responds to the `Request` with a 404 status code and drops the `Request` event.

In this example, the "next controller" for `/questions` is added with the `generate` method. The `generate` method takes a closure that returns an instance of some `RequestController`. Each time the router passes a `Request` a generator, the closure is called, creating a new instance of that `RequestController`, and the `Request` is delivered to that new instance. Here, that new instance is an instance of our `QuestionController`.

We'll get to the specifics of all of that in a moment, but we're at the point that we can run this web server, and that seems more exciting. First, activate the `aqueduct` executable:

```bash
pub global activate aqueduct
```

This command might tell you that `~/.pub-cache/bin` (or some other directory) is not in your `PATH` variable. If that's the case, add it according to the instructions emitted by the command. (For example, if you are on macOS, you'd add `export PATH=$PATH:"~/pub-cache/bin"` to your `~/.bash_profile` and then reload your terminal.)

In the project directory, run the following command to start the application:

```bash
aqueduct serve
```

In a browser, open the URL `http://localhost:8081/questions`. You'll see the list of questions! (You can shut down the server by hitting Ctrl-C in the terminal where you ran `aqueduct serve`.)

Routing and Another Route
---

So far, we've added a route that matches the constant string `/questions`. Routers can do more than match a constant string, they can also include path variables, optional path components, regular expression matching and the wildcard character. We'll add to the existing `/questions` route by allowing requests to get a specific question.

In `quiz.dart`, modify the code in the `QuizRequestSink.setupRouter` by adding "/[:index]" to the route.

```dart
  @override
  void setupRouter(Router router) {
    router
        .route("/questions/[:index]")
        .generate(() => new QuestionController());
  }
```

The square brackets indicate that part of the path is optional, and the colon indicates that it is a path variable. A path variable matches anything. Therefore, this route will match if the path is `/questions` or `/questions/2` or `/questions/foo`.

When using path variables, you may optionally restrict which values they match with a regular expression. The regular expression syntax goes into parentheses after the path variable name. Let's restrict the `index` path variable to only numbers:

```dart
  @override
  void setupRouter(Router router) {
    router
        .route("/questions/[:index(\\d+)]")
        .generate(() => new QuestionController());
  }
```

Now, there are two types of requests that will get forwarded to a `QuestionController` - a request for all questions (`/questions`) and and a request for a specific question at some index (`/questions/1`). We need to add a new responder method to `QuestionController` that gets called when the latter request is made:

```dart
class QuestionController extends HTTPController {
  var questions = [
    "How much wood can a woodchuck chuck?",
    "What's the tallest mountain in the world?"
  ];

  @httpGet
  Future<Response> getAllQuestions() async {
    return new Response.ok(questions);
  }

  @httpGet
  Future<Response> getQuestionAtIndex(@HTTPPath("index") int index) async {
    if (index < 0 || index >= questions.length) {
      return new Response.notFound();
    }

    return new Response.ok(questions[index]);  
  }
}
```

Reload the application by hitting Ctrl-C and then running `aqueduct serve` again. In your browser, enter `http://localhost:8081/questions` and you'll get the list of questions. Then, enter `http://localhost:8081/questions/0` and you'll get the first question. If you enter an index not within the list of questions or something other than an integer, you'll get a 404.

When a `Request` is sent to an `HTTPController`, it evaluates the HTTP method of the request and matches it against every declared responder method. In this case, the `HTTPController` will have two possible choices: `getQuestions` and `getQuestionAtIndex`. From here, it looks at the parameters for each of the methods and the path variables in the `Request`.

Parameters with `HTTPPath` metadata are used to match path variables from the `Request`. If there are no path variables, the no-argument `getAllQuestions` is invoked. If there is one `HTTPPath` argument *and* the name of the path variable is named `index` (the `String` argument to `HTTPPath`), then `getQuestionAtIndex` is called. The name of the path variable is defined by the name of the variable declared in `Router`'s `route` method.

If neither of those scenarios are true, the `HTTPController` responds with 404 and doesn't call any of your responder methods. Because the route is declared to also evaluate a regular expression that restricts `index` to only numeric values, non-numeric values in the `index` portion of the route will also yield a 404. You can try that be hitting `http://localhost:8081/questions/foo` from your browser.

This HTTP method and path variable matching behavior is specific to `HTTPController`.

The More You Know: Multi-threading and Application State
---
In this simple exercise, we used a constant list of question as the source of data for the questions endpoint. For a simple getting-your-feet-wet demo, this is fine.

However, in a real application, it is important that we don't keep any mutable state in a `RequestSink` or any `RequestController`s. This is for three reasons. First, it's just bad practice - web servers should be stateless. They are facilitators between a client and a repository of data, not a repository of data themselves. A repository of data is typically a database.

Second, the way Aqueduct applications are structured makes it really difficult to keep state. For example, `HTTPController` is instantiated each time a new request comes in. Any state they have is discarded after the request is finished processing. This is intentional - you won't run into an issue when scaling to multiple server instances in the future, because the code is already structured to be stateless.

Finally, isolates. Aqueduct applications are set up to run on multiple isolates (the `--isolates` option for the in `aqueduct serve`). An isolate is effectively a thread that shares no memory with other threads. If we were to keep track of state in some way, that state would not be reflected across all of the isolates running on this web server. So depending on which isolate grabbed a request, it may have different state than you might expect. Again, Aqueduct forces you into this model on purpose.

Isolates will spread themselves out across CPUs on the host machine. Each isolate will have its own instance of your `RequestSink` subclass. Having multiple isolates running the same stateless web server on one machine allows for faster request handling. Each isolate also maintains its own set of services, like database connections.

## [Next Chapter: Writing Tests](writing-tests.md)