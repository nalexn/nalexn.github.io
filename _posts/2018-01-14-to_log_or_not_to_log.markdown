---
layout: post
title: To log, or not to log?
description: A better way to use logs in the code
tags: [log,logging,logger,api,log level,programming,ios,development,best practices]
date: 2018-01-14 17:04:44 +0600
comments: true
sharing: true
published: true
img: archive_01.jpg
---

One of the first and basic concepts every developer learns when he kicks off with programming is logging. Being a great alternative to the step by step debugging, an ability to print out an intermediate state of your program' execution is crucial for understanding whether it works as expected.

Were you a Newbie or a seasoned developer, logging is likely to become your only reliable source of truth when you are asked why the rocket you wrote the software for has flown in the opposite direction.

Despite all the benefits, I see the tendency people use logging in their projects less and less often. In this post, I'm going to uncover the potential of logging and restore its reputation. The code examples are in pseudo - Swift code; however, the concept can be taken even if your main programming language is different.

As a quick start, logging can be as simple as that:

```swift
print("Hello, World!")
```

This "raw print out" approach can be taken as you learn the stuff. However, when it comes to the production code, simple _print_ becomes insufficient for addressing all the problems:

* When your user faces an issue using your program, he's unable to easily __export the logs__ and send them over to you for analysis. Quite often, logs are just discarded, so you lose your last chance to get to know what happened.
* Standard _print out_ calls can __slow down__ your program in places where a code chunk executes repeatedly many times per second. This is because _print out_ is set up to collect and include in the log message additional debug information, such as current date and time, executable name, the current phase of the moon, etc.
* As your project grows and _print out_ appears in many places, one day you'll notice that your console output became __overwhelmed__ with inessential information that lulls your vigilance. As a result, when there is an important message sent to the logs, you are likely to miss the notice.

All of this leads the developers to use the third party logging frameworks, which are highly configurable, performance-optimized, and with a better API.

# Logging frameworks

As a consumer of such a framework, the developer mainly cares about the API it provides. For a logging framework you can expect to see something like this:

```swift
log.verbose("Low-level technical information you're not always interested in")
log.debug("Some information helping to debug the program")
log.info("FYI, the flow of the program")
log.warning("A heads up about something starting to go wrong")
log.error("The app faced an error, which might be the outcome of a bug")
```
These diverse calls follow the concept of the __log levels__. The idea is that in your code instead of using `print` here and there you should use either of these methods, explicitly specifying how critical the message is. Later on, by tweaking the framework settings, you get the ability to adjust which messages you want to see in the logs: maybe only _warning_ and _error_ - for production, and all the levels excepting _verbose_ - for development. As a result, all the messages with a lower level that you want are discarded, helping you to manage the logs with ease.

Depending on which logging framework you choose, you can find the following benefits:

* Additional visualization of the log messages in the debug console with <span style="color:red">color</span> or symbols ⚠️ , which attracts attention of the developer when it's needed
* Ability to specify the custom format of the log message for each level - you may only need a timestamp or even nothing but the message itself
* Choosing the log levels you're interested in at the moment, even turning logging off completely if you want to.
* Ability to quickly configure _in one place_ the destination of the logs:
  - Debug console
  - A text file on a disk
  - Local database
  - A remote server (through web API)
* As a bonus to the previous point:
  - Easy export of the logs
  - Advanced search capabilities with GUI for logs in the database or the webserver

# Which level would you use?

There is one significant drawback with using logging frameworks: each time you want to log something you'll have to make a decision which log level to assign to the message, and it can be [not that easy](https://stackoverflow.com/questions/7839565/logging-levels-logback-rule-of-thumb-to-assign-log-levels).
For the error and warning messages this is just obvious, but for _informational_ messages, would you be using _verbose_, _debug_ or _info_, when there is no tangible difference between them?

If you work on a project in a team, you and your teammates may have _different flavor_ to what and how thoroughly to log (_logging depth_). For example, you may prefer to log each step of the algorithm, while the other guy logs only the results of bigger code blocks or functions. But the __worst case__ is when you and your teammate assigns __different log levels for the similar cases__ you worked on independently. This can certainly happen one day if you don't do the code reviews, which eventually can lead you to have long entertaining debug sessions.

The root of this problem is not that one of you was a fool who didn't keep in mind all the written and spoken agreements you had on your project. This happened because __the programming environment you had in the project was too liberal; it promoted the ambiguity of choice__. In software engineering, you want to avoid ambiguity whenever possible, right? So, does it sound like we should have used just _error_, _warning_, and _info_ levels? Actually not - this wouldn't solve the problem of having different preferred logging depth, which I mentioned above.

There has to be a better way.

# Eliminating the choice

In the real-life of human beings, we all value the freedom of choice. But I'm convinced the same does not apply to the software development. When your programming environment provides you with a dozen different ways to do the same thing - your chances to shoot yourself in the foot increase dramatically.
Life becomes much easier when you don't have to make a choice - so let's take it away!

Instead of making developer guess `"To log, or not to log?"`, we can introduce more descriptive logging API that prompts the developer when and what to log:

```swift
class Logger {

  // Private reference to a third party logger instance
  private let loggerInstance = MyFavouriteLogger()

  func initialization(subject: Any, result: Result) {
    switch result {
    case .success:
      // In case initialization of the "subject" was successful, we use log level 'info'
      loggerInstance.info("[Success] Initialization of \(subject.class)")
    case .failure(error):
      // Another log level in case of an error
      loggerInstance.error("[Failure] Initialization of \(subject.class)")
    }
  }

  func httpRequest(request: URLRequest) {
    // Specific request details are quite low level, so we can use level 'verbose'
    loggerInstance.verbose("Request: \(request.httpMethod) \(request.url) \(request.body)")
  }

  // Don't afraid to read the code - it's self descriptive
  func httpResponse(response: URLResponse, request: URLRequest, result: Result) {
    switch result {
    case .success:
      loggerInstance.verbose("[Success] Response: \(response.httpStatusCode) \(response.body)")
    case .failure(error):
      loggerInstance.error("[Failure] Response: \(response.httpStatusCode) \(response.body) Error: \(error)")
    }
  }

  func viewDidLoad(viewController: UIViewController) {
    loggerInstance.debug("\(viewController.class) has finished loading view")
  }
}
```
Now with this API instead of __asking how__ to log, we __stipulate what and where__ to log. Moreover, the log message format, as well as the log level, are now stored in one place, which allows us to comply with the [DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself) principle.

Here is how we'd use this logger from the client code:

```swift
let log = Logger()
```
```swift
// Networking
func loadPlayers() {
  let request = httpClient.GET("/players")
  log.httpRequest(request)
  httpClient.sendRequest(request) { response, error in
    log.httpResponse(response, request, Result(error))
    // handle the response
  }
}
```
```swift
// Launching any service
try {
  let audioSession = AudioSession()
  audioSession.setup()
  log.initialization(AudioSession.self, result: .success)
} catch {
  log.initialization(AudioSession.self, result: .failure(error))
}
```
```swift
// You'll know which screens were visited and when
class MyViewController: UIViewController {
  func viewDidLoad() {
    super.viewDidLoad()
    log.viewDidLoad(self)
    // other code
  }
}
```
# Logging + Analytics

The fact that we've introduced a Facade for logging made it super easy to add any additional functionality behind it.

Say you want to integrate Analytics into your app.
Traditionally, you'd implement a wrapper for the library (Google Analytics, for example), and then spent a fair amount of time checking all the places in your project inserting the same line of code that records an event with Analytics.

Nevermore!

If you think about it, Analytics is yet another logger! So why would you expose these intimate details that you're using Analytics to the client code - let it only know that there is `Logger`, which does all the magic.

All you need to do is to update the implementation of the `Logger` to report only those events you need to the Analytics selectively. Yay!