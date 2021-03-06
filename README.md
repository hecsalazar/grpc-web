## Overview

gRPC-Web provides a Javascript library that lets browser clients access a gRPC
service. You can find out much more about gRPC in its own
[website](https://grpc.io).

The current release is a Beta release, and we expect to announce
General-Availability by Oct. 2018.

gRPC-Web clients connect to gRPC services via a special gateway proxy: the
current version of the library uses [Envoy](https://www.envoyproxy.io/) by
default, in which gRPC-Web support is built-in.

In the future, we expect gRPC-Web to be supported in language-specific Web
frameworks, such as Python, Java, and Node. See the
[roadmap](https://github.com/grpc/grpc-web/blob/master/ROADMAP.md) doc.


## Quick Demo

Try gRPC-Web and run a quick Echo example from the browser!

From the repo root directory:

```sh
$ docker-compose pull prereqs common echo-server envoy commonjs-client
$ docker-compose up -d echo-server envoy commonjs-client
```

Open a browser tab, and go to:

```
http://localhost:8081/echotest.html
```

To shutdown: `docker-compose down`.

## Runtime Library

The gRPC-Web runtime library is available at `npm`:

```sh
$ npm i grpc-web
```


## Code Generator Plugin

You can compile the `protoc-gen-grpc-web` protoc plugin from this repo:

```sh
$ sudo make install-plugin
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
in TypeScript.


### Wire Format Mode

For more information about the gRPC-Web wire format, please see the
[specification](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-WEB.md#protocol-differences-vs-grpc-over-http2)
here.

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

Let's take a look at how gRPC-Web works with a simple example. You can find out
how to build, run and explore the example yourself in
[Build and Run the Echo Example](net/grpc/gateway/examples/echo).


### 1. Define your service

The first step when creating any gRPC service is to define it. Like all gRPC
services, gRPC-Web uses
[protocol buffers](https://developers.google.com/protocol-buffers/) to define
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
builds a simple C++ gRPC backend server and the Envoy proxy.

For the Echo service: see the
[service implementations](https://github.com/grpc/grpc-web/blob/master/net/grpc/gateway/examples/echo/echo_service_impl.cc).

For the Envoy proxy: see the
[config yaml file](https://github.com/grpc/grpc-web/blob/master/net/grpc/gateway/examples/echo/envoy.yaml).


### 3. Write your JS client

Once the server and gateway are up and running, you can start making gRPC calls
from the browser!

Create your client

```js
var echoService = new proto.mypackage.EchoServiceClient(
  'http://localhost:8080');
```

Make a unary RPC call

```js
var request = new proto.mypackage.EchoRequest();
request.setMessage(msg);
var metadata = {'custom-header-1': 'value1'};
var call = echoService.echo(request, metadata, function(err, response) {
  if (err) {
    console.log(err.code);
    console.log(err.messge);
  } else {
    console.log(response.getMessage());
  }
});
call.on('status', function(status) {
  console.log(status.code);
  console.log(status.details);
  console.log(status.metadata);
});
```

Server-side streaming is supported!

```js
var stream = echoService.serverStreamingEcho(streamRequest, metatada);
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
```

You can find a more in-depth tutorial from
[this page](https://github.com/grpc/grpc-web/blob/master/net/grpc/gateway/examples/echo/tutorial.md).

## TypeScript Support

The `grpc-web` module can now be imported as a TypeScript module. This is
currently an experimental feature. Any feedback welcome!

When using the `protoc-gen-grpc-web` protoc plugin, mentioned above, pass in
either:

 - `import_style=commonjs+dts`: existing CommonJS style stub + `.d.ts` typings
 - `import_style=typescript`: full TypeScript output

```ts
import * as grpcWeb from 'grpc-web';
import {EchoServiceClient} from './echo_grpc_web_pb';
import {EchoRequest, EchoResponse} from './echo_pb';

const echoService = new EchoServiceClient('http://localhost:8080', null, null);

const request = new EchoRequest();
request.setMessage('Hello World!');

const call = echoService.echo(request, {'custom-header-1': 'value1'},
  (err: grpcWeb.Error, response: EchoResponse) => {
    console.log(response.getMessage());
  });
call.on('status', (status: grpcWeb.Status) => {
  // ...
});
```

See a full TypeScript example
[here](https://github.com/grpc/grpc-web/blob/master/net/grpc/gateway/examples/echo/ts-example/client.ts).

## Proxy Interoperability

Multiple proxies supports the gRPC-Web protocol. Currently, the default proxy
is [Envoy](https://www.envoyproxy.io), which supports gRPC-Web out of the box.

```sh
$ docker-compose up -d echo-server envoy commonjs-client
```

An alternative is to build Nginx that comes with this repository.

```sh
$ docker-compose up -d echo-server nginx commonjs-client
```

You can also try this
[gRPC-Web Go Proxy](https://github.com/improbable-eng/grpc-web/tree/master/go/grpcwebproxy).

```sh
$ docker-compose up -d echo-server grpcwebproxy binary-client
```

## Acknowledgement

Big thanks to the following contributors for making significant contributions to
this project!

* [zaucy](https://github.com/zaucy)
* [yannic](https://github.com/yannic)
