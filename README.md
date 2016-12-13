protobuf.js
===========
[![travis][travis-image]][travis-url] [![npm][npm-dl-image]][npm-url] [![npm][npm-image]][npm-url] [![donate][paypal-image]][paypal-url]

**Protocol Buffers** are a language-neutral, platform-neutral, extensible way of serializing structured data for use in communications protocols, data storage, and more, originally designed at Google ([see](https://developers.google.com/protocol-buffers/)).

**protobuf.js** is a pure JavaScript implementation for node and the browser. It efficiently encodes plain objects and custom classes and works out of the box with .proto files.

[travis-image]: https://img.shields.io/travis/dcodeIO/protobuf.js.svg
[travis-url]: https://travis-ci.org/dcodeIO/protobuf.js
[npm-image]: https://img.shields.io/npm/v/protobufjs.svg
[npm-url]: https://npmjs.org/package/protobufjs
[npm-dl-image]: https://img.shields.io/npm/dm/protobufjs.svg
[paypal-image]: https://img.shields.io/badge/paypal-donate-yellow.svg
[paypal-url]: https://www.paypal.com/cgi-bin/webscr?cmd=_donations&business=dcode%40dcode.io&item_name=%3C3%20protobuf.js

**Recommended read:** [Changes in protobuf.js 6.0](https://github.com/dcodeIO/protobuf.js/wiki/Changes-in-protobuf.js-6.0)

Features
--------
* Optimized [for performance](#performance)
* Exhaustive [browser support](#compatibility)
* Managed [TypeScript definitions](#usage-with-typescript)
* Elaborate [API documentation](#documentation)
* Convenient [CLI utilities](#command-line)
* Seamless [browserify integration](#browserify-integration)

Contents
--------

* [Usage](#usage)<br />
  How to include protobuf.js in your project.

* [Examples](#examples)<br />
  A few examples to get you started.

* [Module Structure](#module-structure)<br />
  A brief introduction to the structure of the exported module.

* [Documentation](#documentation)<br />
  A list of available documentation resources.

* [Command line](#command-line)<br />
  How to use the command line utility.

* [Building](#building)<br />
  How to build the library and its components yourself.

* [Performance](#performance)<br />
  A few internals and a benchmark on performance.

* [Compatibility](#compatibility)<br />
  Notes on compatibility regarding browsers and optional libraries.

Usage
---------------

### node.js

```
$> npm install protobufjs
```

```js
var protobuf = require("protobufjs");
```

### Browsers

Development:
```
<script src="//cdn.rawgit.com/dcodeIO/protobuf.js/6.1.0/dist/protobuf.js"></script>
```

Production:
```
<script src="//cdn.rawgit.com/dcodeIO/protobuf.js/6.1.0/dist/protobuf.min.js"></script>
```

**NOTE:** Remember to replace the version tag with the exact [release](https://github.com/dcodeIO/protobuf.js/releases) your project depends upon.

Or [download](https://github.com/dcodeIO/protobuf.js/tree/master/dist) the library.

The `protobuf` namespace will always be available globally but also supports AMD.

Examples
--------

### Using .proto files

```protobuf
// awesome.proto
package awesomepackage;
syntax = "proto3";

message AwesomeMessage {
    string awesome_field = 1; // becomes awesomeField
}
```

```js
protobuf.load("awesome.proto", function(err, root) {
    if (err) throw err;

    // Obtain a message type
    var AwesomeMessage = root.lookup("awesomepackage.AwesomeMessage");

    // Create a new message
    var message = AwesomeMessage.create({ awesomeField: "AwesomeString" });

    // Encode a message
    var buffer = AwesomeMessage.encode(message).finish();
    // ... do something with buffer

    // Or, encode a plain object
    var buffer = AwesomeMessage.encode({ awesomeField: "AwesomeString" }).finish();
    // ... do something with buffer

    // Decode a buffer
    var message = AwesomeMessage.decode(buffer);
    // ... do something with message

    // If your application uses length-delimited buffers, there is also encodeDelimited and decodeDelimited.
});
```

You can also use promises by omitting the callback:

```js
protobuf.load("awesome.proto")
    .then(function(root) {
       ...
    });
```

### Using reflection only

```js
...
var Root  = protobuf.Root,
    Type  = protobuf.Type,
    Field = protobuf.Field;

var AwesomeMessage = new Type("AwesomeMessage").add(new Field("awesomeField", 1, "string"));

var root = new Root().define("awesomepackage").add(AwesomeMessage);

// Continue at "Create a new message" above
...
```

### Using custom classes

```js
...

function AwesomeMessage(properties) {
    protobuf.Message.call(this, properties);
}
protobuf.Class.create(root.lookup("awesomepackage.AwesomeMessage") /* or use reflection */, AwesomeMessage);

var message = new AwesomeMessage({ awesomeField: "AwesomeString" });

// Continue at "Encode a message" above
```

Custom classes are automatically populated with static `encode`, `encodeDelimited`, `decode`, `decodeDelimited` and `verify` methods and reference their reflected type via the `$type` property. Note that there are no methods (just `$type`) on instances by default as method names might conflict with field names.

### Using services

```protobuf
// greeter.proto
service Greeter {
    rpc SayHello (HelloRequest) returns (HelloReply) {}
}

message HelloRequest {
    string name = 1;
}

message HelloReply {
    string message = 1;
}
```

```js
...
var Greeter = root.lookup("Greeter");
var greeter = Greeter.create(rpcImpl, false, false); // rpcImpl (see below), requestDelimited?, responseDelimited?

greeter.sayHello({ name: 'you' }, function(err, response) {
    console.log('Greeting:', response.message);
});
```

To make this work, all you have to do is provide an `rpcImpl`, which is an asynchronous function that takes the reflected service method, the binary HelloRequest and a node-style callback as its parameters. For example:

```js
function rpcImpl(method, requestData, callback) {
    // perform the request using an HTTP request or a WebSocket for example
    var responseData = ...;
    // and call the callback with the binary response afterwards:
    callback(null, responseData);
}
```

There is also an [example for streaming RPC](https://github.com/dcodeIO/protobuf.js/blob/master/examples/streaming-rpc.js).

### Usage with TypeScript

```ts
/// <reference path="node_modules/protobufjs/types/protobuf.js.d.ts" />

import * as protobuf from "protobufjs";
...
```

Module Structure
----------------
The library exports a flat `protobuf` namespace including but not restricted to the following members, ordered by category:

### Parser

* **load(filename: `string|string[]`, [root: `Root`], [callback: `function(err: Error, [root: Root])`]): `Promise|undefined`** [[source](https://github.com/dcodeIO/protobuf.js/blob/master/src/index.js)]<br />
  Loads one or multiple .proto or preprocessed .json files into a common root namespace.

* **loadSync(filename: `string|string[]`, [root: `Root`]): `Root`** [[source](https://github.com/dcodeIO/protobuf.js/blob/master/src/index.js)]<br />
  Synchronously loads one or multiple .proto or preprocessed .json files into a common root namespace (node only).

* **parse(source: `string`): `Object`** [[source](https://github.com/dcodeIO/protobuf.js/blob/master/src/parse.js)]<br />
  Parses the given .proto source and returns an object with the parsed contents.

### Serialization

* **Writer** [[source](https://github.com/dcodeIO/protobuf.js/blob/master/src/writer.js)]<br />
  Wire format writer using `Uint8Array` if available, otherwise `Array`.

* **Reader** [[source](https://github.com/dcodeIO/protobuf.js/blob/master/src/reader.js)]<br />
  Wire format reader using `Uint8Array` if available, otherwise `Array`.

### Reflection

* **Namespace** _extends **ReflectionObject**_ [[source](https://github.com/dcodeIO/protobuf.js/blob/master/src/namespace.js)]<br />
  Base class of all reflection objects containing nested objects.

* **Root** _extends **Namespace**_ [[source](https://github.com/dcodeIO/protobuf.js/blob/master/src/root.js)]<br />
  Root namespace.

* **Type** _extends **Namespace**_ [[source](https://github.com/dcodeIO/protobuf.js/blob/master/src/type.js)]<br />
  Reflected message type.

* **Field** _extends **ReflectionObject**_ [[source](https://github.com/dcodeIO/protobuf.js/blob/master/src/field.js)]<br />
  Reflected message field.

* **MapField** _extends **Field**_ [[source](https://github.com/dcodeIO/protobuf.js/blob/master/src/mapfield.js)]<br />
  Reflected message map field.

* **Enum** _extends **ReflectionObject**_ [[source](https://github.com/dcodeIO/protobuf.js/blob/master/src/enum.js)]<br />
  Reflected enum.

* **Service** _extends **Namespace**_ [[source](https://github.com/dcodeIO/protobuf.js/blob/master/src/service.js)]<br />
  Reflected service.

* **Method** _extends **ReflectionObject**_ [[source](https://github.com/dcodeIO/protobuf.js/blob/master/src/method.js)]<br />
  Reflected service method.

### Runtime

* **Class** [[source](https://github.com/dcodeIO/protobuf.js/blob/master/src/class.js)]<br />
  Runtime class providing the tools to create your own custom classes.

* **Message** [[source](https://github.com/dcodeIO/protobuf.js/blob/master/src/message.js)]<br />
  Abstract runtime message.

### Utility

* **util** [[source](https://github.com/dcodeIO/protobuf.js/blob/master/src/util.js)]<br />
  Various utility functions.

For less common members, see the API documentation.

Documentation
-------------

* [Google's Developer Guide](https://developers.google.com/protocol-buffers/docs/overview)

* [protobuf.js API Documentation](http://dcode.io/protobuf.js/)

Command line
------------

The `pbjs` command line utility can be used to bundle and translate between .proto and .json files.

```
Consolidates imports and converts between file formats.

  -t, --target    Specifies the target format. Also accepts a path to require a custom target.

                  json          JSON representation
                  json-module   JSON representation as a module (AMD, CommonJS, global)
                  proto2        Protocol Buffers, Version 2
                  proto3        Protocol Buffers, Version 3
                  static        Static code without reflection
                  static-module Static code without reflection as a module (AMD, CommonJS, global)

  -p, --path      Adds a directory to the include path.

  -o, --out       Saves to a file instead of writing to stdout.

  -w, --wrap      Specifies the wrapper to use for *-module targets. Also accepts a path.

                  default   Default wrapper supporting both CommonJS and AMD
                  commonjs  CommonJS only wrapper
                  amd       AMD only wrapper

  -r, --root      Specifies an alternative protobuf.roots name for *-module targets.

usage: pbjs [options] file1.proto file2.json ...
```

For production environments it is recommended to bundle all your .proto files to a single .json file, which reduces the number of network requests and parser invocations required:

```
$> pbjs -t json file1.proto file2.proto > bundle.json
```

Now, either include this file in your final bundle:

```js
var root = protobuf.Root.fromJSON(require("./bundle.json"));
```

or load it the usual way:

```js
protobuf.load("bundle.json", function(err, root) {
    ...
});
```

### Generating TypeScript definitions from static modules

Likewise, the `pbts` command line utility can be used to generate TypeScript definitions from `pbjs`-generated static modules.

```
Generates TypeScript definitions from annotated JavaScript files.

  -n, --name      Specifies the module name.

  -o, --out       Saves to a file instead of writing to stdout.

usage: pbts [options] file1.js file2.js ...
```

### Descriptors vs. static modules

While .proto and JSON files require the full library (about 18kb gzipped), pretty much all code but the relatively short descriptors is shared.

Static code, on the other hand, requires just the minimal runtime (about 5.5kb gzipped), but generates relatively large code bases without any reflection features.

When `new Function` is supported (and it usually is), there is no difference performance-wise as the code generated statically is the same generated at runtime.

Building
--------

To build the library or its components yourself, clone it from GitHub and install the development
dependencies:

```
$> git clone https://github.com/dcodeIO/protobuf.js.git
$> cd protobuf.js
$> npm install --dev
```

Building the development and production versions with their respective source maps to `dist/`:

```
$> npm run build
```

Building the documentation to `docs/`:

```
$> npm run docs
```

Building the TypeScript definition to `types/`:

```
$> npm run types
```

### Browserify integration

protobuf.js integrates into any browserify build-process. There are a few possible tweaks:

* If performance is a concern or IE8 support is required, you should make sure to exclude the browserified `buffer` module and let protobuf.js do its thing with Uint8Array/Array instead.
* If you do not need int64 support, you can explicitly exclude the `long` module. It will be included otherwise.
* If your application does not rely on the following modules and package size is a concern, you can also exclude `process` , `_process` and `fs`.
* If you have any special requirements, there is [the bundler](https://github.com/dcodeIO/protobuf.js/blob/master/scripts/bundle.js) as a reference.

Performance
-----------
The package includes a benchmark that tries to compare performance to native JSON as far as this is possible. On an i7-2600K running node 6.9.1 it yields:

```
benchmarking encoding performance ...

Type.encode to buffer x 481,172 ops/sec ±0.48% (92 runs sampled)
JSON.stringify to string x 307,509 ops/sec ±1.04% (92 runs sampled)
JSON.stringify to buffer x 164,463 ops/sec ±1.37% (89 runs sampled)

      Type.encode to buffer was fastest
   JSON.stringify to string was 36.4% slower
   JSON.stringify to buffer was 66.1% slower

benchmarking decoding performance ...

Type.decode from buffer x 1,319,810 ops/sec ±0.71% (92 runs sampled)
JSON.parse from string x 298,578 ops/sec ±0.98% (90 runs sampled)
JSON.parse from buffer x 267,471 ops/sec ±0.81% (89 runs sampled)

    Type.decode from buffer was fastest
     JSON.parse from string was 77.4% slower
     JSON.parse from buffer was 79.8% slower

benchmarking combined performance ...

Type to/from buffer x 262,728 ops/sec ±0.92% (92 runs sampled)
JSON to/from string x 129,405 ops/sec ±0.78% (94 runs sampled)
JSON to/from buffer x 89,523 ops/sec ±0.71% (89 runs sampled)

        Type to/from buffer was fastest
        JSON to/from string was 50.7% slower
        JSON to/from buffer was 65.9% slower

benchmarking verifying performance ...

Type.verify x 5,833,382 ops/sec ±0.98% (85 runs sampled)

                Type.verify was fastest
```

Note that JSON is a native binding nowadays and as such is about as fast as it possibly can get. So, how can protobuf.js be faster?

* The benchmark is [somewhat flawed](https://github.com/dcodeIO/protobuf.js/blob/master/bench/index.js).
* Reader and writer interfaces configure themselves according to the environment to eliminate redundant conditionals.
* Node-specific reader and writer subclasses benefit from node's buffer binding.
* Reflection has built-in code generation that builds type-specific encoders, decoders and verifiers at runtime.
* Encoders and decoders do not verify that required fields are present (with proto3 this is dead code anyway). There is a `verify` method to check this manually instead - where applicable.
* For entirely bogus values encoders intentionally rely on runtime errors to be thrown somewhere down the road.
* Quite a bit of V8-specific profiling is accountable for everything else.

Note that code generation requires `new Function(...)` (basically `eval`) support and that an equivalent but slower fallback will be used where unsupported.

You can also run [the benchmark](https://github.com/dcodeIO/protobuf.js/blob/master/bench/index.js) ...

```
$> npm run bench
```

and [the profiler](https://github.com/dcodeIO/protobuf.js/blob/master/bench/prof.js) yourself (the latter requires a recent version of node):

```
$> npm run prof <encode|decode|encode-browser|decode-browser> [iterations=10000000]
```

Note that as of this writing, the benchmark suite performs significantly slower on node 7.2.0 compared to 6.9.1 because moths.

Compatibility
-------------

[![Sauce Test Status](https://saucelabs.com/browser-matrix/protobuf.svg)](https://saucelabs.com/u/protobuf)

* Because the internals of this package do not rely on `google/protobuf/descriptor.proto`, options are parsed and presented literally.
* If typed arrays are not supported by the environment, plain arrays will be used instead.
* Support for pre-ES5 environments like IE8 can be achieved by [using a polyfill](https://github.com/dcodeIO/protobuf.js/blob/master/scripts/polyfill.js) and, instead of using property getters and setters on reflection objects, calling the respective functions prefixed with `get`, `set` or `is` directly (i.e. calling `Type#getFieldsById()` instead of accessing `Type#fieldsById`).
* If you need a proper way to work with 64 bit values (uint64, int64 etc.), you can install [long.js](https://github.com/dcodeIO/long.js) alongside this library. All 64 bit numbers will then be returned as a `Long` instance instead of a possibly unsafe JavaScript number ([see](https://github.com/dcodeIO/long.js)).

**License:** [Apache License, Version 2.0](http://www.apache.org/licenses/LICENSE-2.0.html), bundled external libraries may have [their own license](https://github.com/dcodeIO/protobuf.js/tree/master/lib)

[![Analytics](https://ga-beacon.appspot.com/UA-40277577-2/dcodeIO/protobuf.js)](https://github.com/igrigorik/ga-beacon)
