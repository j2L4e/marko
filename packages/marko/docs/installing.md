# Installation

## Trying out Marko

If you just want to play around with Marko in the browser, head on over to our [Try Online](https://markojs.com/try-online) feature. You'll be able to develop a Marko application right in your browser.

## Creating new apps

If you're starting from scratch, you can use Marko's [cli](https://github.com/marko-js/cli) commands to quickly create a starter app:

```bash
npx @marko/create
```

## Direct usage

### Installing

The Marko compiler runs on [Node.js](https://nodejs.org/) and can be installed using [npm](https://www.npmjs.com/package/marko):

```
npm install marko
```

or using [yarn](https://yarnpkg.com):

```
yarn add marko
```

### In the browser

Let's say we have a simple view that we want to render in the browser: `hello.marko`

_hello.marko_

```marko
<h1>Hello ${input.name}</h1>
```

First, let's create a `client.js` that requires the view and renders it to the body:

_client.js_

```js
var helloComponent = require("./hello");

helloComponent.renderSync({ name: "Marko" }).appendTo(document.body);
```

We will also create a barebones HTML page to host our application:

_index.html_

```
<!doctype html>
<html>
<head>
    <title>Marko Example</title>
</head>
<body>

</body>
</html>
```

Now, we need to bundle these files for use in the browser. We can use a tool called [`lasso`](https://github.com/lasso-js/lasso) to do that for us, so let's get it (and the marko plugin) installed:

```
npm install --global lasso-cli
npm install --save lasso lasso-marko
```

Now we can build our bundle for the browser:

```
lasso --main client.js --plugins lasso-marko --inject-into index.html
```

This builds a `client.js` file to the newly created `static/` directory and injects the required `<script>` tags into our HTML page to load our application in the browser. If we had css in the view then `<link>` tags would have also been added.

Load up that page in your browser and you should see `Hello Marko` staring back at you.

### On the server

#### Require Marko views

Marko provides a custom Node.js require extension that allows you to `require` Marko views exactly like a standard JavaScript module. Take the following example `server.js`:

_hello.marko_

```marko
<div>
    Hello ${input.name}!
</div>
```

_server.js_

```js
// The following line installs the Node.js require extension
// for `.marko` files.  This should be called once near the start
// of your application before requiring any `*.marko` files.
require("marko/node-require");

var fs = require("fs");

// Load a Marko view by requiring a .marko file:
var hello = require("./hello");
var out = fs.createWriteStream("hello.html", { encoding: "utf8" });
hello.render({ name: "Frank" }, out);
```

Using the Node.js require extension is completely optional. If you prefer to not use the Node.js require extension then you will need to precompile all of the marko templates using [Marko CLI](https://github.com/marko-js/cli):

```bash
marko compile hello.marko
```

This will produce a `hello.marko.js` file next to the original template. The generated `.js` file will be what gets loaded by the Node.js runtime. It is important to leave off the `.marko` extension when requiring a Marko template so that the `.js` will be resolved correctly.

If you wish to only use the require extension in development, you can conditionally require it.

```js
if (!process.env.NODE_ENV) {
  require("marko/node-require");
}
```

#### Serving a simple page

Let's update `server.js` to serve the view from an http server:

_server.js_

```js
// Allow requiring `.marko` files
require("marko/node-require");

var http = require("http");
var hello = require("./hello");
var port = 8080;

http
  .createServer((req, res) => {
    // let the browser know html is coming
    res.setHeader("content-type", "text/html");

    // render the output to the `res` output stream
    hello.render({ name: "Marko" }, res);
  })
  .listen(port);
```

And give `hello.marko` some content:

_hello.marko_

```marko
<h1>Hello ${input.name}</h1>
```

Start the server (`node server.js`) and open your browser to [http://localhost:8080](http://localhost:8080) where you should see the heading `Hello Marko`.

#### Initializing server-rendered components

Marko automatically injects a list of components that need to be mounted in the browser, right before the closing `</body>` tag (as such, it required that you include a `<body>` in your rendered output).

However, you still need to bundle the CSS & JavaScript for your page and include the proper `link`, `style`, and `script` tags. Luckily, the `lasso` taglib will do all the heavy lifting for you.

First install `lasso` and `lasso-marko`:

```
npm install --save lasso lasso-marko @lasso/marko-taglib
```

Next, register and config the lasso:

_server.js_

```js
var isProduction = process.env.NODE_ENV === "production";

// Configure lasso to control how JS/CSS/etc. is delivered to the browser
require("lasso").configure({
  plugins: [
    "lasso-marko" // Allow Marko templates to be compiled and transported to the browser
  ],
  outputDir: __dirname + "/static", // Place all generated JS/CSS/etc. files into the "static" dir
  bundlingEnabled: isProduction, // Only enable bundling in production
  minify: isProduction, // Only minify JS and CSS code in production
  fingerprintsEnabled: isProduction // Only add fingerprints to URLs in production
});
```

Next, in your page or layout view, add the `lasso-head` and `lasso-body` tags:

_layout.marko_

```marko
<!doctype>
<html>
<head>
    <title>Hello world</title>
    <lasso-head/>
</head>
<body>
    <${input.body}/>
    <lasso-body/>
</body>
</html>
```

Finally, configure your server to serve the static files that `lasso` generates:

If you're using `express`, just do:

_server.js_

```js
app.use(require("lasso/middleware").serveStatic());
```

And if `koa`, do:

_server.js_

```js
const mount = require("koa-mount");
const serve = require("koa-static");

app.use(mount("/static", serve(__dirname + "/static")));
```

For the full application code for the Koa and assets bundling, please see the sample: [Marko + Koa](https://github.com/marko-js-samples/marko-koa).
