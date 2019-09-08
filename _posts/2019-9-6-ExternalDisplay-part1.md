---
layout: post
title: External display with flutter / iOS part 1
---

TL;DR See modification in [AppDelegate.swift](https://github.com/oqu/external_display/blob/P1-4/ios/Runner/AppDelegate.swift) and [main.dart](https://github.com/oqu/external_display/blob/P1-4/lib/main.dart)


## Why am I looking into external display support for Flutter

My kids are practicing [Abacus and Flash Anzan](https://en.wikipedia.org/wiki/Mental_abacus).

They already have an application for Flash Anzan to practice on their own but I thought of something different that I can implement with Flutter:
* Use external display to present questions/answers/score
* Use main display (phone/tablet) to select time, number of digit, ...
* Multi-player mode where devices can be used to enter answers

After some research, I did not find something already working for Flutter with external display. This is a good opportunity to learn something !

## Reading

I started by creating a simple flutter app with vscode and opened the ios project on Xcode.
I read the following articles:
* [Displaying Content on a Connected Screen from Apple documentation](https://developer.apple.com/documentation/uikit/windows_and_screens/displaying_content_on_a_connected_screen)
* [External Display Support on iOS by Geoff Hackworth](https://medium.com/@hacknicity/external-display-support-on-ios-665cd1774511)


## Create the application from template

First, I created the project:
``` bash
flutter create --org io.github.oqu -i swift -a kotlin \
  --description 'external display adventures' external_display
```

You can also get the source code from [github](https://github.com/oqu/external_display)

Running the app on iOS simulator with external display yields the following:

[iOS Screen]({{site.baseurl}}/images/p1-1-0.png) |  [External display]({{site.baseurl}}/images/p1-1-1.png)
:-------------------------:|:-------------------------:
![]({{site.baseurl}}/images/p1-1-0.png)  |  ![]({{site.baseurl}}/images/p1-1-1.png)

## First attempt : failure

Following iOS development center documentation, I edited ios/Runner/AppDelegate.swift to add the required observers.

I created a ExternalDisplay class like so:
``` swift
class ExternalDisplay {
  var additionalWindows = [UIWindow]()

  func setup() {
    NotificationCenter.default.addObserver(forName: .UIScreenDidConnect,
        object: nil, queue: nil) { notification in
      // Get the new screen information.
      let newScreen = notification.object as! UIScreen
      let screenDimensions = newScreen.bounds

      // Configure a window for the screen.
      let newWindow = UIWindow(frame: screenDimensions)
      newWindow.screen = newScreen

      // You must show the window explicitly.
      newWindow.isHidden = false
      // Save a reference to the window in a local array.
      self.additionalWindows.append(newWindow)
      // Create a FlutterViewController.
      // There was several constructor to pick from ...
      let extVC = FlutterViewController()
      newWindow.rootViewController = extVC
    }
    NotificationCenter.default.addObserver(forName:
      .UIScreenDidDisconnect,
                                           object: nil,
                                           queue: nil) { notification in
      let screen = notification.object as! UIScreen

      // Remove the window associated with the screen.
      for window in self.additionalWindows {
        if window.screen == screen {
          // Remove the window and its contents.
          let index = self.additionalWindows.index(of: window)
          self.additionalWindows.remove(at: index!)
        }
      }
    }
  }
}
```


When trying to create the flutter view controller, code completion showed

![]({{site.baseurl}}/images/fvc-constructor.png)

This is where I made the mistake to look into FlutterDartProject.<BR>
I end up trying to create a new FlutterDart project by modifying iOS project.<BR>
I tried to create a new bundle (similar to App.framework) by manually building another flutter project.


## Success : display the same view

Using the simple constructor allowed to display the same view as the phone.<BR>

![]({{site.baseurl}}/images/p1-extdisp-success.png)

See [github repo with tag P1-2](https://github.com/oqu/external_display/tree/P1-2) for full code.


## Display a different view

Because the external display does not have input, I need to present a different view on the external display.
This can be achieved by defining the initial route the flutter view controller should display

``` swift
// modify the setup function to 
func setup(route : String) {
  // ...
  let extVC = FlutterViewController()
  extVC.setInitialRoute(route)
  // ...
}

// Also, in AppDelegate, define a route:
extDisplay.setup(route:"/external")
```

Main application in dart needs to define a widget to display for this route:
``` dart
class ExternalDisplay extends StatefulWidget {
  @override
  _ExternalDisplayState createState() => _ExternalDisplayState();
}

class _ExternalDisplayState extends State<ExternalDisplay> {

  int counter = 0;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Container(child: Center(child: Text("External counter: $counter"),),),);
  }
}

// Modify the main app to handle /external route
  home: MyHomePage(title: 'External display adventure'),
  routes: {
    "/external" : (context) =>  ExternalDisplay(),
    },
```

See full code at [github repo with tag P1-2](https://github.com/oqu/external_display/tree/P1-2).<BR>
This is what the external display looks like with this simple widget.
![]({{site.baseurl}}/images/p1-extdisp-diffv.png)


## Modify External content display from main screen

I want to modify the value of the counter of the external display.<BR>
How do I do that ?<BR>

Looking at dart observatory:

![]({{site.baseurl}}/images/p1-diffv-isolates.png)

There is 2 isolates. One for the main screen and one for the external display.

How can we exchange information or event ?
* Web services
* Platform channel

Lets try platform channels !

### Dart code

When the user press the FAB, we want to send an event to the external display

``` dart

// First import flutter services to access PlatformChannel
import 'package:flutter/services.dart';

// Define the name of the function to be used by both MyApp and ExternalDisplay
const MethodSetCounter = "setCounter";

// Modify _MyHomePageState to call a function to set the counter value
static const platform = const MethodChannel('io.github.oqu/externalA');

void _incrementCounter() {
  platform.invokeMethod(MethodSetCounter,[_counter + 1]);
  setState(() {
    _counter++;
  });
}

// Modify _ExternalDisplayState to add the handler
@override
void didChangeDependencies() {
  super.didChangeDependencies();
  platform.setMethodCallHandler((message) async {
    if (message.method == MethodSetCounter) {
      var c = getFirstInteger(message.arguments);
      if (c != null) {
        setState(() {
        counter = c;
      });
      }
    }
    return "ok";
  });
}
// Type checking to retrieve the first argument as integer
int getFirstInteger(dynamic args) {
  if (args is! List<dynamic>) {
    return null;
  }
  var ld = args as List<dynamic>;
  if (ld.length > 0) {
    if (ld[0] is int) {
      return (ld[0] as int);
    }
  }
  return null;
}
```

### Swift code

AppDelegate needs to be modified to routes method call to all external displays.
``` swift
// Modify setup function to add the main flutter view controller as argument : 
func setup(route: String, mainFvc: FlutterViewController) {
  // ...
  let counterChannel = FlutterMethodChannel(name: "io.github.oqu/externalA", binaryMessenger:     mainFvc.binaryMessenger)

  counterChannel.setMethodCallHandler {
    (call: FlutterMethodCall, _: @escaping FlutterResult) -> Void in
    // Dispatch event to all external displays
    for window in self.additionalWindows {
      let fvc = window.rootViewController as! FlutterViewController
      // fvc.invokeMethod()
      let c = FlutterMethodChannel(name: "io.github.oqu/externalB", binaryMessenger: fvc.binaryMessenger)
      c.invokeMethod(call.method, arguments: call.arguments)
    }
  }
}

// Modify the way it is initialized
let controller: FlutterViewController = window?.rootViewController as! FlutterViewController
    extDisplay.setup(route: "/external", mainFvc: controller)
```

<BR>
The two screens are now in sync !


[iOS Screen]({{site.baseurl}}/images/p1-4-0.png) |  [External display]({{site.baseurl}}/images/p1-4-1.png)
:-------------------------:|:-------------------------:
![]({{site.baseurl}}/images/p1-4-0.png)  |  ![]({{site.baseurl}}/images/p1-4-1.png)

Thanks for reading.
