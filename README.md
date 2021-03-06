# @small-tech/express-ws [(link to upstream of this fork)](https://github.com/aral/express-ws)

__This fork uses native Node `require`s (CommonJS) instead of [upstream](https://github.com/HenningM/express-ws)’s ES6 imports (ESM).__

Please note that I’ve contributed all changes in this fork (apart from the ESM → CommonJS changes) back upstream in the following unmerged [pull request](https://github.com/HenningM/express-ws/pull/122).

Here’s why [I](https://ar.al) hacked together this fork:

  - I needed to include [my changes](https://github.com/HenningM/express-ws/pull/122) from my git repository.
  - To do so, I needed to build it and [the build process wasn’t documented](https://github.com/HenningM/express-ws/issues/123).
  - I don’t want to use Babel and add a build process for a simple library.
  - Without Babel it was causing errors in [Nexe](https://github.com/nexe/nexe) due to the additional complexity in module loading.
  - ES6 imports in Node.js… Y, tho? :)

If these are not concerns for you, and if you don’t need references to the WebSocket instance from your routes or broadcast client filtering (“room” functionality), please [head on over to the upstream](https://github.com/HenningM/express-ws). If you `npm install` it instead of including it from source, the issues I outlined above should not affect you.

## Install

```shell
npm install @small-tech/express-ws
```

## Details

In addition to using `require()` instead of ES6 imports, this fork also enables you to access the WebSocket Server instance and the Express app instance from within routes via `this`:

```js
app.ws('/broadcast', function(ws, req) {

  ws.on('message', message => {
    this.getWss().clients.forEach(client => {
      client.send(message)
    })
  })
})
```
Note that if you have multiple web socket routes, the above example will broadcast the message to all clients, not just those connected to the `/broadcast` route.

Note that this list will include *all* clients, not just those for a specific route - this means that it's often *not* a good idea to use this for broadcasts, for example. For broadcasts, use the `setRoom()` and `broadcast()` methods as shown in the [chat example](examples/chat.js):

```javascript
const express = require('express');
const expressWs = require('express-ws')(express());

const app = expressWs.app;

function roomHandler(client, request) {
  client.room = this.setRoom(request);
  console.log(`New client connected to ${client.room}`);

  client.on('message', (message) => {
    const numberOfRecipients = this.broadcast(client, message);
    console.log(`${client.room} message broadcast to ${numberOfRecipients} recipient${numberOfRecipients === 1 ? '' : 's'}.`);
  });
}

app.ws('/room1', roomHandler);
app.ws('/room2', roomHandler);

app.listen(3000, () => {
  console.log('\nChat server running on http://localhost:3000\n\nFor Room 1, connect to http://localhost:3000/room1\nFor Room 2, connect to http://localhost:3000/room2\n');
});
```

## A note on route scope

Routes are bound to the wsInstance so you can access `.getWss()`, `.setRoom()`, `.broadcast()` and `.app` via `this` in your routes even if the original wsInstance is not in scope (e.g., if you have your routes defined in external files).

## Development

This module is written in ES6 and uses native Node.js requires (CommonJS) unlike upstream which uses ESM. Among other things, it means that you can wrap apps that use this module into native binaries using [Nexe](https://github.com/nexe/nexe).

## License

Commits up to and including 8efedd5d0946f23c7e386ce44586a7e384a1635c are licensed under BSD-2-Clause by the original author. Commits from and including 18a8d15ef8e63e601ee723d09fd435dd1ee2bed9 are licensed under AGPL version 3.0 or later.

__We now return to the regular upstream documentation…__

---

[WebSocket](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API) endpoints for [Express](http://expressjs.com/) applications. Lets you define WebSocket endpoints like any other type of route, and applies regular Express middleware. The WebSocket support is implemented with the help of the [ws](https://github.com/websockets/ws) library.

## Installation

`npm install --save express-ws`

## Usage

__Full documentation can be found in the API section below. This section only shows a brief example.__

Add this line to your Express application:

```javascript
var expressWs = require('express-ws')(app);
```

__Important: Make sure to set up the `express-ws` module like above *before* loading or defining your routers!__ Otherwise, `express-ws` won't get a chance to set up support for Express routers, and you might run into an error along the lines of `router.ws is not a function`.

After setting up `express-ws`, you will be able to add WebSocket routes (almost) the same way you add other routes. The following snippet sets up a simple echo server at `/echo`.  The `ws` parameter is an instance of the WebSocket class described [here](https://github.com/websockets/ws/blob/master/doc/ws.md#class-websocket).

```javascript
app.ws('/echo', function(ws, req) {
  ws.on('message', function(msg) {
    ws.send(msg);
  });
});
```

It works with routers, too, this time at `/ws-stuff/echo`:

```javascript
const router = express.Router();

router.ws('/echo', function(ws, req) {
  ws.on('message', function(msg) {
    ws.send(msg);
  });
});

app.use("/ws-stuff", router);
```

## Full example

```javascript
const express = require('express');
const app = express();
const expressWs = require('express-ws')(app);

app.use(function (req, res, next) {
  console.log('middleware');
  req.testing = 'testing';
  return next();
});

app.get('/', function(req, res, next){
  console.log('get route', req.testing);
  res.end();
});

app.ws('/', function(ws, req) {
  ws.on('message', function(msg) {
    console.log(msg);
  });
  console.log('socket', req.testing);
});

app.listen(3000);
```

## API

### expressWs(app, *server*, *options*)

Sets up `express-ws` on the specified `app`. This will modify the global Router prototype for Express as well - see the `leaveRouterUntouched` option for more information on disabling this.

* __app__: The Express application to set up `express-ws` on.
* __server__: *Optional.* When using a custom `http.Server`, you should pass it in here, so that `express-ws` can use it to set up the WebSocket upgrade handlers. If you don't specify a `server`, you will only be able to use it with the server that is created automatically when you call `app.listen`.
* __options__: *Optional.* An object containing further options.
  * __leaveRouterUntouched:__ Set this to `true` to keep `express-ws` from modifying the Router prototype. You will have to manually `applyTo` every Router that you wish to make `.ws` available on, when this is enabled.
  * __wsOptions:__ Options object passed to WebSocketServer constructor. Necessary for any ws specific features.

This function will return a new `express-ws` API object, which will be referred to as `wsInstance` in the rest of the documentation.

### wsInstance.app

This property contains the `app` that `express-ws` was set up on.

### wsInstance.getWss()

Returns the underlying WebSocket server/handler. You can use `wsInstance.getWss().clients` to obtain a list of all the connected WebSocket clients for this server.

Note that this list will include *all* clients, not just those for a specific route - this means that it's often *not* a good idea to use this for broadcasts, for example. For broadcasts, use the `setRoom()` and `broadcast()` methods as shown in the [chat example](examples/chat.js):

```javascript
const express = require('express');
const expressWs = require('express-ws')(express());

const app = expressWs.app;

function roomHandler(client, request) {
  client.room = this.setRoom(request);
  console.log(`New client connected to ${client.room}`);

  client.on('message', (message) => {
    const numberOfRecipients = this.broadcast(client, message);
    console.log(`${client.room} message broadcast to ${numberOfRecipients} recipient${numberOfRecipients === 1 ? '' : 's'}.`);
  });
}

app.ws('/room1', roomHandler);
app.ws('/room2', roomHandler);

app.listen(3000, () => {
  console.log('\nChat server running on http://localhost:3000\n\nFor Room 1, connect to http://localhost:3000/room1\nFor Room 2, connect to http://localhost:3000/room2\n');
});
```

### wsInstance.applyTo(router)

Sets up `express-ws` on the given `router` (or other Router-like object). You will only need this in two scenarios:

1. You have enabled `options.leaveRouterUntouched`, or
2. You are using a custom router that is not based on the express.Router prototype.

In most cases, you won't need this at all.

### A note on route scope

Routes are bound to the wsInstance so you can access `.getWss()`, `.setRoom()`, `.broadcast()` and `.app` via `this` in your routes even if the original wsInstance is not in scope (e.g., if you have your routes defined in external files).

## Development

This module is written in ES6 and uses ESM.

