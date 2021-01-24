<!--
 * @Description: 
 * @Author: ekibun
 * @Date: 2020-08-08 08:16:50
 * @LastEditors: ekibun
 * @LastEditTime: 2020-10-03 00:44:41
-->
# flutter_qjs

![Pub](https://img.shields.io/pub/v/flutter_qjs.svg)
![Test](https://github.com/ekibun/flutter_qjs/workflows/Test/badge.svg)

This plugin is a simple js engine for flutter using the `quickjs` project with `dart:ffi`. Plugin currently supports all the platforms except web!

## Getting Started

### Basic usage

Firstly, create a `FlutterQjs` object, then call `dispatch` to dispatch event loop:

```dart
final engine = FlutterQjs(
  stackSize: 1024 * 1024, // change stack size here.
);
engine.dispatch();
```

Use `evaluate` method to run js script, now you can use it synchronously, or use await to resolve `Promise`:

```dart
try {
  print(engine.evaluate(code ?? ''));
} catch (e) {
  print(e.toString());
}
```

Method `close` can destroy quickjs runtime that can be recreated again if you call `evaluate`. Parameter `port` should be close to stop `dispatch` loop when you do not need it.

```dart
engine.port.close(); // stop dispatch loop
engine.close();      // close engine
engine = null;
```

Data conversion between dart and js are implemented as follow:

| dart      | js                 |
| --------- | ------------------ |
| Bool      | boolean            |
| Int       | number             |
| Double    | number             |
| String    | string             |
| Uint8List | ArrayBuffer        |
| List      | Array              |
| Map       | Object             |
| Function  | function(....args) |
| Future    | Promise            |
| Object    | DartObject         |

**notice:** Dart function parameter `thisVal` is used to store `this` in js.

### Use modules

ES6 module with `import` function is supported and can be managed in dart with `moduleHandler`:

```dart
final engine = FlutterQjs(
  moduleHandler: (String module) {
    if(module == "hello")
      return "export default (name) => `hello \${name}!`;";
    throw Exception("Module Not found");
  },
);
```

then in JavaScript, `import` function is used to get modules:

```javascript
import("hello").then(({default: greet}) => greet("world"));
```

**notice:** Module handler should be called only once for each module name. To reset the module cache, call `FlutterQjs.close` then `evaluate` again.

To use async function in module handler, try [Run on isolate thread](#Run-on-isolate-thread)

### Run on isolate thread

Create a `IsolateQjs` object, pass handlers to resolving modules. Async function such as `rootBundle.loadString` can be used now to get modules:

```dart
dynamic methodHandler(String method, List arg) {
  switch (method) {
    case "http":
      return Dio().get(arg[0]).then((response) => response.data);
    default:
      throw Exception("No such method");
  }
}
final engine = IsolateQjs(
  methodHandler: methodHandler,
  moduleHandler: (String module) async {
    return await rootBundle.loadString(
        "js/" + module.replaceFirst(new RegExp(r".js$"), "") + ".js");
  },
);
// not need engine.dispatch();
```

Same as run on main thread, use `evaluate` to run js script. In this way, `Promise` return by `evaluate` will be automatically tracked and return the resolved data:

```dart
try {
  print(await engine.evaluate(code ?? ''));
} catch (e) {
  print(e.toString());
}
```

Method `close` can destroy quickjs runtime that can be recreated again if you call `evaluate`.

**notice:** Make sure arguments passed to `IsolateJSFunction` are avaliable for isolate, such as primities and top level function. Method `bind` can help to pass instance function to isolate:

```dart
await setToGlobalObject("func", await engine.bind(() {
  // DO SOMETHING
}))
```

[This example](example/lib/main.dart) contains a complete demonstration on how to use this plugin.

## Breaking change in v0.3.0

`channel` function is no longer utilized by default.
Use js function to set to global:

```dart
final setToGlobalObject = await engine.evaluate("(key, val) => this[key] = val;");
setToGlobalObject("channel", methodHandler);
```