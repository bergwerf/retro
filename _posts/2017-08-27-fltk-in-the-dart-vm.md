---
layout:      post
title:       FLTK in the Dart VM
date:        2017-08-27
categories:  posts
author:      Herman Bergwerf
---
About a year ago I wrote some code that makes is possible to develop a native
GUI using FLTK and the Dart programming language. The code can be executed in
the standard Dart VM using an extension that I developed for this purpose. I
have been wanting to write a short post about this, so here it finally is.

In this post I will briefly discuss the implementation of the extension and the
Dart code on top of it. Then I will give some examples of what you can do with
this library. Finally I will introduce a small program I wrote to 'hot reload'
an FLTK app. Note that all of the stuff presented here is experimental and not
being developed any further. It is not deemed suitable to be used for any
production projects. The library discussed here can be found on [GitHub][3].

## Dart VM extension
FLTK, the [Fast Light Toolkit][0], is a C++ library for developing native GUIs.
Some years ago I used this library to learn building applications in C++ (among
other things, I used it to write a simple sound synthesizer). I decided it would
be an interesting experiment to try to integrate this fairly simple and
straightforward library with the Dart VM. To do this an extension can be
developed in C and loaded as a shared object. More details can be found on
[dartlang.org][1]. The API header file is also a good place to look for
information: [dart_api.h][2].

One of the things I usually try to avoid most is redundancy, for this reason I
wrote a rather messy code generation program that consumes a YAML file that
describes FLTK classes, their methods, arguments, types, enums, etc. I use
[Mustache][5] templates for code generation. The detailed code can be found in
[tool/codegen/generate.dart][4]. The generated code basically contains a single
function for every method that we want to call from Dart. In Dart we can specify
a 'native' function and indicate a string that can be used to lookup the
function symbol in the shared object. In this function some wrapping and
unwrapping of Dart variables is going on. Dart allows the storage of pointers in
Dart variable handles (in so called native instance fields).

## Custom widget drawing in Dart
One of the first challenges was implementing custom widget drawing directly in
Dart. This is one of the things that works very nicely in FLTK. For example,
consider the following C++ code.

```cpp
#include <FL/Fl.H>
#include <FL/fl_draw.H>
#include <FL/Fl_Box.H>
#include <FL/Fl_Double_Window.H>

// Widget that draws two diagonal lines
class XWidget : public Fl_Widget {
 public:
  // Constuctor
  XWidget(int x, int y, int w, int h) : Fl_Widget(x, y, w, h, 0) {}

  // Draws the lines
  void draw() {
    fl_color(FL_BLACK);
    int x1 = x(), y1 = y();
    int x2 = x() + w() - 1, y2 = y() + h() - 1;
    fl_line(x1, y1, x2, y2);
    fl_line(x1, y2, x2, y1);
  }
};

int main() {
  auto window = new Fl_Double_Window(200, 200, "X");
  auto x = new XWidget(0, 0, window->w(), window->h());
  window->resizable(x);
  window->show();
  return Fl::run();
}
```

To make a similar mechanism in Dart, I pass a function pointer to the C++
function, and call it every time the draw function is executed (a function from
the Dart API called `Dart_Invoke` must be used). A Dart program like the
following one can now be written.

```dart
import 'package:fltk/fltk.dart' as fl;

/// Widget that draws two diagonal lines
class XWidget extends fl.Widget {
  XWidget(int x, int y, int w, int h) : super(x, y, w, h);

  /// Draws the lines
  void draw() {
    fl.color = fl.BLACK;
    int x1 = this.x(), y1 = this.y();
    int x2 = this.x() + this.w() - 1, y2 = this.y() + this.h() - 1;
    fl.line(x1, y1, x2, y2);
    fl.line(x1, y2, x2, y1);
  }
}

int main() {
  var window = new fl.DoubleWindow(200, 200, 'X');
  var x = new XWidget(0, 0, window.w(), window.h());
  window.resizable = x;
  window.show();
  return fl.run();
}
```

## Hooking into the FLTK event system
The next step is to handle events. FLTK offers two main ways to do this. You can
pass a function pointer to the `Fl_Widget.callback` method. This function will
be executed based on `Fl_Widget.when`. I will not explain the details here, if
you really want to know how it works you can look through the [FLTK docs][6].
The second method is to override the `Fl_Widget.handle` method which recieves
an integer whenever an event happens indicating the event. Other information
such as the mouse coordinates can be retrieved through global FLTK functions.

To pass a Dart closure to `Fl_Widget.callback` I used a feature in FLTK that
allows you to bind a user data pointer to the callback. This way only a single
callback handling function is needed in C++ (you can find it in the generated
source under the name `callback_wrapper`). To override the handle method in Dart
I used a similar method as described for the draw method in the previous
section. Overriding the handle method for other widgets than the base widget
(`Fl_Widget`) is not a good idea because this would conflict with the event
handling implemented for this widget. Therefore the callback closure must be
used to receive events from built-in widgets (e.g. `Fl_Button`) and the handle
method should only be used for custom widgets.

## Delivering events via streams
Next I wanted to use streams rather than 'boring' closures to handle events. A
major issue for this is that the FLTK draw/event cycle basically blocks the Dart
thread. As a result using anything that relies on the Dart task scheduler does
not work, including asynchronous streams. A solution is using synchronized
streams only. As a result a program like the following one can be written.

```dart
import 'package:fltk/fltk.dart' as fl;

int main() {
  fl.useSyncStreams = true;
  fl.scheme = 'gleam';

  var window = new fl.Window(300, 200, 'Click the button...');
  var button = new fl.Button(0, 0, window.w(), window.h(), 'ON');
  button.onCallback.listen((_) {
    button.label = button.label == 'ON' ? 'OFF' : 'ON';
  });
  window.end();
  window.show();

  return fl.run();
}
```

I did find a solution to this problem by replacing a call to `Fl::run` with
a periodic timer in Dart that uses `Fl::check`, a function provided by FLTK that
is intended to allow FLTK to check for queued events. The resulting run function
looks like this.

```dart
Future<int> runAsync([Duration interval = Duration.ZERO]) {
  final completer = new Completer<int>();
  _PeriodicTimerCallback cycle;
  cycle = (Timer timer) {
    final state = check();
    // When state equals 0, all windows were closed.
    if (state == 0) {
      timer.cancel();
      completer.complete(state);
    }
  };

  new Timer.periodic(interval, cycle);
  return completer.future;
}
```

This function allows writing the previous callback example using asynchronous
streams like in the example below. Unfortunately this results in extremely high
CPU usage, and I have been unable to find a solution. Increasing the timer
interval reduces this to some extent, but not to a reasonable level.

```dart
import 'dart:async';
import 'package:fltk/fltk.dart' as fl;

Future<int> main() {
  fl.useSyncStreams = false;
  fl.scheme = 'gleam';

  var window = new fl.Window(300, 200, 'Click the button...');
  var button = new fl.Button(0, 0, window.w(), window.h(), 'ON');
  button.onCallback.listen((_) {
    button.label = button.label == 'ON' ? 'OFF' : 'ON';
  });
  window.end();
  window.show();

  return fl.runAsync();
}
```

## Working with OpenGL and Cairo
Something that is quite exiting is the ability to directly use Cairo and OpenGL
in FLTK. This is also possible via Dart using the `dartgl` and the `cairodart`
packages from Pub which provide native bindings to these libraries. FLTK can
setup the required context. As a result a simple drawing program can be written
as in the following example. A number of other examples can be found in
[test/classic][7].

```dart
import 'dart:math';
import 'package:cairodart/cairodart.dart';
import 'package:fltk/fltk.dart' as fl;

int main() {
  final window = new fl.DoubleWindow(720, 480, 'Dart');
  final surface = new fl.CairoSurface(0, 0, window.w(), window.h());

  int x, y, xo, yo;
  final blue = new Random().nextDouble();
  final ctx = surface.getContext();
  ctx
    ..setSourceRgb(1, 1, 1)
    ..paint()
    ..setSourceRgb(0, 0, 0)
    ..lineWidth = 5.0
    ..lineJoin = LineJoin.Round;

  surface.addHandler((event, data) {
    switch (event) {
      case fl.Event.PUSH:
        // Update position.
        x = data.x;
        y = data.y;
        xo = x;
        yo = y;
        return true;

      case fl.Event.DRAG:
        // Draw line segment
        ctx
          ..moveTo(xo, yo)
          ..lineTo(x, y)
          ..lineTo(data.x, data.y)
          ..setSourceRgb(x / surface.w(), y / surface.h(), blue)
          ..stroke();
        surface.redraw();

        // Update position.
        xo = x;
        yo = y;
        x = data.x;
        y = data.y;
        return true;

      default:
        return false;
    }
  });

  window.show();
  return fl.run();
}
```

## Hot reload
At the Dart Developer Summit 2016 I learned about the Dart VM service protocol
and the ability to trigger a hot reload. I created a small program that watches
for file changes in the source directory and reloads the app as a result. To
make this work I wrote a small UI building system and function to merge new and
old widget trees. As a result a program does not have to be closed in order to
update styles or callback functions. An example program is shown below. I also
uploaded a video to YouTube where I demonstrate this ability. You can find the
video [here][8].

```dart
import 'dart:async';
import 'package:fltk/flfn.dart' as fl;

fl.App buildApp() => new fl.App(windows: {
      // Application window
      'main': new fl.Window(300, 200, 'Click the button...', children: {
        // First button
        'btn-one': new fl.Button(0, 0, 150, 200, label: '1', box: fl.UP_BOX,
            callback: (app, button) {
          // Clicking the first button manipulates the second button.
          final btn = app.getButton('main/btn-two');
          btn.label = btn.label == 'ON' ? 'OFF' : 'ON';
        }),
        // Second button
        'btn-two': new fl.Button(150, 0, 150, 200, label: 'ON', box: fl.UP_BOX,
            callback: (app, button) {
          // Clicking the second button manipulates the first button.
          final btn = app.getButton('main/btn-one');
          btn.label = btn.label == '1' ? '0' : '1';
        })
      })
    }, scheme: 'plastic');

Future<int> main() async {
  return fl.runApp(buildApp);
}
```

I hope you enjoyed reading this. Feel free to send me an email if you have
further questions or remarks.

[0]: http://www.fltk.org
[1]: https://www.dartlang.org/articles/dart-vm/native-extensions
[2]: https://github.com/dart-lang/sdk/blob/master/runtime/include/dart_api.h
[3]: https://github.com/bergwerf/fltk-dart
[4]: https://github.com/bergwerf/fltk-dart/blob/master/tool/codegen/generate.dart
[5]: https://mustache.github.io/
[6]: http://www.fltk.org/doc-1.3/index.html
[7]: https://github.com/bergwerf/fltk-dart/tree/master/test/classic
[8]: https://www.youtube.com/watch?v=JeHjfvoUXEA
