# gRPC Web &middot; [![npm version](https://img.shields.io/npm/v/grpc-web.svg?style=flat)](https://www.npmjs.com/package/grpc-web)

A JavaScript implementation of [gRPC][] for browser clients. For more information,
including a **quick start**, see the [gRPC-web documentation][grpc-web-docs].

gRPC-web clients connect to gRPC services via a special proxy; by default,
gRPC-web uses [Envoy][].

In the future, we expect gRPC-web to be supported in language-specific web
frameworks for languages such as Python, Java, and Node. For details, see the
[roadmap](doc/roadmap.md).

## Streaming Support
gRPC-web currently supports 2 RPC modes:
- Unary RPCs ([example](#make-a-unary-rpc-call))
- Server-side Streaming RPCs ([example](#server-side-streaming)) (NOTE: Only when [`grpcwebtext`](#wire-format-mode) mode is used.)

Client-side and Bi-directional streaming is not currently supported (see [streaming roadmap](doc/streaming-roadmap.md)).

## Quick Start

Eager to get started? Try the [Hello World example][]. From this example, you'll
learn how to do the following:

 - Define your service using protocol buffers
 - Implement a simple gRPC Service using NodeJS
 - Configure the Envoy proxy
 - Generate protobuf message classes and client service stub for the client
 - Compile all the JS dependencies into a static library that can be consumed
   by the browser easily

## Advanced Demo: Browser Echo App

You can also try to run a more advanced Echo app from the browser with a
streaming example.

From the repo root directory:

```sh
$ docker-compose pull prereqs node-server envoy commonjs-client
$ docker-compose up node-server envoy commonjs-client
```

Open a browser tab, and visit http://localhost:8081/echotest.html.

To shutdown: `docker-compose down`.

## Runtime Library

The gRPC-web runtime library is available at `npm`:

```sh
$ npm i grpc-web
```

## Code Generator Plugin

You can download the `protoc-gen-grpc-web` protoc plugin from our
[release](https://github.com/grpc/grpc-web/releases) page:

If you don't already have `protoc` installed, you will have to download it
first from [here](https://github.com/protocolbuffers/protobuf-javascript/releases).

> **NOTE:** Javascript output is no longer supported by `protocolbuffers/protobuf` package as it previously did. Please use the releases from [protocolbuffers/protobuf-javascript](https://github.com/protocolbuffers/protobuf-javascript/releases) instead.

Make sure they are both executable and are discoverable from your PATH.

For example, in MacOS, you can do:

```
$ sudo mv ~/Downloads/protoc-gen-grpc-web-1.4.2-darwin-x86_64 \
    /usr/local/bin/protoc-gen-grpc-web
$ chmod +x /usr/local/bin/protoc-gen-grpc-web
```

## Client Configuration Options

Typically, you will run the following command to generate the proto messages
and the service client stub from your `.proto` definitions:

```sh
$ protoc -I=$DIR echo.proto \
    --js_out=import_style=commonjs:$OUT_DIR \
    --grpc-web_out=import_style=commonjs,mode=grpcwebtext:$OUT_DIR
```

You can then use Browserify, Webpack, Closure Compiler, etc. to resolve imports
at compile time.

### Import Style

`import_style=closure`: The default generated code has
[Closure](https://developers.google.com/closure/library/) `goog.require()`
import style.

`import_style=commonjs`: The
[CommonJS](https://requirejs.org/docs/commonjs.html) style `require()` is
also supported.

`import_style=commonjs+dts`: (Experimental) In addition to above, a `.d.ts`
typings file will also be generated for the protobuf messages and service stub.

`import_style=typescript`: (Experimental) The service stub will be generated
in TypeScript. See **TypeScript Support** below for information on how to
generate TypeScript files.

**Note: The `commonjs+dts` and `typescript` styles are only supported by
`--grpc-web_out=import_style=...`, not by `--js_out=import_style=...`.**

### Wire Format Mode

For more information about the gRPC-web wire format, see the
[specification](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-WEB.md#protocol-differences-vs-grpc-over-http2).

`mode=grpcwebtext`: The default generated code sends the payload in the
`grpc-web-text` format.

  - `Content-type: application/grpc-web-text`
  - Payload are base64-encoded.
  - Both unary and server streaming calls are supported.

`mode=grpcweb`: A binary protobuf format is also supported.

  - `Content-type: application/grpc-web+proto`
  - Payload are in the binary protobuf format.
  - Only unary calls are supported for now.

## How It Works

Let's take a look at how gRPC-web works with a simple example. You can find out
how to build, run and explore the example yourself in
[Build and Run the Echo Example](net/grpc/gateway/examples/echo).

### 1. Define your service

The first step when creating any gRPC service is to define it. Like all gRPC
services, gRPC-web uses
[protocol buffers](https://developers.google.com/protocol-buffers) to define
its RPC service methods and their message request and response types.

```protobuf
message EchoRequest {
  string message = 1;
}

...

service EchoService {
  rpc Echo(EchoRequest) returns (EchoResponse);

  rpc ServerStreamingEcho(ServerStreamingEchoRequest)
      returns (stream ServerStreamingEchoResponse);
}
```

### 2. Run the server and proxy

Next you need to have a gRPC server that implements the service interface and a
gateway proxy that allows the client to connect to the server. Our example
builds a simple Node gRPC backend server and the Envoy proxy.

For the Echo service: see the
[service implementations](net/grpc/gateway/examples/echo/node-server/server.js).

For the Envoy proxy: see the
[config yaml file](net/grpc/gateway/examples/echo/envoy.yaml).

### 3. Write your JS client

Once the server and gateway are up and running, you can start making gRPC calls
from the browser!

Create your client:

```js
var echoService = new proto.mypackage.EchoServiceClient(
  'http://localhost:8080');
```

#### Make a unary RPC call:

```js
var request = new proto.mypackage.EchoRequest();
request.setMessage(msg);
var metadata = {'custom-header-1': 'value1'};
echoService.echo(request, metadata, function(err, response) {
  if (err) {
    console.log(err.code);
    console.log(err.message);
  } else {
    console.log(response.getMessage());
  }
});
```

#### Server-side streaming:

```js
var stream = echoService.serverStreamingEcho(streamRequest, metadata);
stream.on('data', function(response) {
  console.log(response.getMessage());
});
stream.on('status', function(status) {
  console.log(status.code);
  console.log(status.details);
  console.log(status.metadata);
});
stream.on('end', function(end) {
  // stream end signal
});

// to close the stream
stream.cancel()
```

For an in-depth tutorial, see [this
page](net/grpc/gateway/examples/echo/tutorial.md).

## Setting Deadline

You can set a deadline for your RPC by setting a `deadline` header. The value
should be a Unix timestamp, in milliseconds.

```js
var deadline = new Date();
deadline.setSeconds(deadline.getSeconds() + 1);

client.sayHelloAfterDelay(request, {deadline: deadline.getTime().toString()},
  (err, response) => {
    // err will be populated if the RPC exceeds the deadline
    ...
  });
```

## TypeScript Support

The `grpc-web` module can now be imported as a TypeScript module. This is
currently an experimental feature. Any feedback welcome!

When using the `protoc-gen-grpc-web` protoc plugin, mentioned above, pass in
either:

 - `import_style=commonjs+dts`: existing CommonJS style stub + `.d.ts` typings
 - `import_style=typescript`: full TypeScript output

Do *not* use `import_style=typescript` for `--js_out`, it will silently be
ignored. Instead you should use `--js_out=import_style=commonjs`, or
`--js_out=import_style=commonjs,binary` if you are using `mode=grpcweb`. The
`--js_out` plugin will generate JavaScript code (`echo_pb.js`), and the
`-grpc-web_out` plugin will generate a TypeScript definition file for it
(`echo_pb.d.ts`). This is a temporary hack until the `--js_out` supports
TypeScript itself.

For example, this is the command you should use to generate TypeScript code
using the binary wire format

```sh
$ protoc -I=$DIR echo.proto \
  --js_out=import_style=commonjs,binary:$OUT_DIR \
  --grpc-web_out=import_style=typescript,mode=grpcweb:$OUT_DIR
```

It will generate the following files:

* `EchoServiceClientPb.ts` - Generated by `--grpc-web_out`, contains the
TypeScript gRPC-web code.
* `echo_pb.js` - Generated by `--js_out`, contains the JavaScript Protobuf
code.
* `echo_pb.d.ts` - Generated by `--grpc-web_out`, contains TypeScript
definitions for `echo_pb.js`.

### Using Callbacks

```ts
import * as grpcWeb from 'grpc-web';
import {EchoServiceClient} from './EchoServiceClientPb';
import {EchoRequest, EchoResponse} from './echo_pb';

const echoService = new EchoServiceClient('http://localhost:8080', null, null);

const request = new EchoRequest();
request.setMessage('Hello World!');

const call = echoService.echo(request, {'custom-header-1': 'value1'},
  (err: grpcWeb.RpcError, response: EchoResponse) => {
    console.log(response.getMessage());
  });
call.on('status', (status: grpcWeb.Status) => {
  // ...
});
```

(See [here](https://github.com/grpc/grpc-web/blob/4d7dc44c2df522376394d3e3315b7ab0e010b0c5/packages/grpc-web/index.d.ts#L29-L39) full list of possible `.on(...)` callbacks)

### (Option) Using Promises (Limited features)

NOTE: It is not possible to access the `.on(...)` callbacks (e.g. for `metadata` and `status`) when Promise is used.

```ts
this.echoService.echo(request, {'custom-header-1': 'value1'})
  .then((response: EchoResponse) => {
    console.log(`Received response: ${response.getMessage()}`);
  }).catch((err: grpcWeb.RpcError) => {
    console.log(`Received error: ${err.code}, ${err.message}`);
  });
```

For the full TypeScript example, see
[ts-example/client.ts](net/grpc/gateway/examples/echo/ts-example/client.ts) with the [instructions](net/grpc/gateway/examples/echo/ts-example) to run.

## Ecosystem

### Proxy Interoperability

Multiple proxies support the gRPC-web protocol.

1. The current **default proxy** is [Envoy][], which supports gRPC-web out of the box.

	```sh
	$ docker-compose up -d node-server envoy commonjs-client
	```

2. You can also try the [gRPC-web Go proxy][].

	```sh
	$ docker-compose up -d node-server grpcwebproxy binary-client
	```

3. Apache [APISIX](https://apisix.apache.org/) has also added grpc-web support, and more details can be found [here](https://apisix.apache.org/blog/2022/01/25/apisix-grpc-web-integration/).

4. [Nginx](https://www.nginx.com/) has a grpc-web module ([doc](https://nginx.org/en/docs/http/ngx_http_grpc_module.html), [announcement](https://www.nginx.com/blog/nginx-1-13-10-grpc/))), and seems to work with simple configs, according to user [feedback](https://github.com/grpc/grpc-web/discussions/1322).

### Web Frameworks with gRPC-Web support
- [Armeria (JVM)](https://armeria.dev/docs/server-grpc/#grpc-web)
- [Tonic (Rust)](https://docs.rs/tonic-web/latest/tonic_web/)

[Envoy]: https://www.envoyproxy.io
[gRPC]: https://grpc.io
[grpc-web-docs]: https://grpc.io/docs/languages/web
[gRPC-web Go Proxy]: https://github.com/improbable-eng/grpc-web/tree/master/go/grpcwebproxy
[Hello World example]: net/grpc/gateway/examples/helloworld
