<div align="center">
    <h1>Koa SSLify</h1>
    <a href="https://travis-ci.org/turboMaCk/koa-sslify">
        <img src="https://travis-ci.org/turboMaCk/koa-sslify.svg?branch=master" alt="build">
    </a>
    <a href="https://codeclimate.com/github/turboMaCk/koa-sslify">
      <img src="https://codeclimate.com/github/turboMaCk/koa-sslify/badges/gpa.svg" alt="code climate">
    </a>
    <a href="https://badge.fury.io/js/koa-sslify">
      <img src="https://badge.fury.io/js/koa-sslify.svg" alt="version">
    </a>
    <p>Enforce HTTPS middleware for Koa.js</p>
</div>

[Koa.js](http://koajs.com/) middleware to enforce HTTPS connection on any incoming requests.
In case of a non-encrypted HTTP request, koa-sslify automatically redirects to an HTTPS address using a `301 permanent redirect`
(or optionally `302 Temporary Redirect`).

Koa SSLify can also work behind reverse proxies (load balancers) like on Heroku, Azure, GCP Ingress etc
and supports custom implementations of proxy resolvers.

## Install

```
$ npm install --save koa-sslify
```

## Usage

Importing default factory function:

```js
const sslify = require('koa-sslify').default; // factory with default options
const Koa = require('koa');

app = new Koa();
app.use(sslify());
```

Default function accepts several options.

| Name                      | Type          | Default           | Description                                          |
|---------------------------|---------------|-------------------|------------------------------------------------------|
| `resolver`                | Function      | `httpsResolver`   | Function used to test if request is secure           |
| `hostname`                | String        | `undefined`       | Hostname for redirect (uses request host if not set) |
| `port`                    | Integer       | `443`             | Port of HTTPS server                                 |
| `ignoreUrl`               | Boolean       | `false`           | Ignore url path (redirect to domain)                 |
| `temporary`               | Boolean       | `false`           | Temporary mode (use 302 Temporary Redirect)          |
| `skipDefaultPort`         | Boolean       | `true`            | Avoid `:403` port in redirect url                    |
| `redirectMethods`         | Array<String> | `['GET', 'HEAD']` | Whitelist methods that should be redirected          |
| `internalRedirectMethods` | Array<String> | `[]`              | Whitelist methods for `307 Internal Redirect`        |
| `disallowStatus`          | Integer       | `405`             | Status returned for dissalowed methods               |

### Resolvers

Resolver is a function from classic Koa `ctx` object to boolean.
This function is used to determine if request is or is not secured (true means is secure).
Middlware calls this function and based on its returned value either passes
controll to next middleware or responds to the request with appropriete redirect response.

There are several resolvers provided by this library but it should be very easy to implement
any type of custom check as well.

for instance Heroku has reverse proxy that uses `x-forwarded-proto` header.
This is how you can configure app with this resolver:

```js
const {
  // middlware factory
  default: sslify,
  // resolver needed
  resolver: xForwardedProtoResolver
} = require('koa-sslify');
const Koa = require('koa');

app = new Koa();
// init middlware with resolver
app.use(sslify(resolver));
```

Those are all resolver provided by default:

| Name                        | Used by                                                                                                                 | Example                                   |
|-----------------------------|-------------------------------------------------------------------------------------------------------------------------|-------------------------------------------|
| `httpsResolver`             | Node.js server running with tls support                                                                                 | `sslify()`                                |
| `xForwardedProtoResolver    | Heroku, Google Ingress, Nodejitsu                                                                                       | `sslify(xForwardedProtoResolver)`         |
| `azureResolver`             | Azure                                                                                                                   | `sslify(azureResolver)`                   |
| `customProtoHeaderResolver` | any non-standard implementation (Kong)                                                                                  | `sslify(customProtoHeader('x-protocol'))` |
| `forwardedResolver`         | [standard header](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Destructuring_assignment) | `sslify(forwardedResolver)                |

Some additianal ifromation about reverse proxies:

#### Reverse Proxies (Heroku, Nodejitsu, GCE Ingress and others)

Heroku, nodejitsu, GCE Ingress and other hosters often use reverse proxies which offer SSL endpoints
but then forward unencrypted HTTP traffic to the website.
This makes it difficult to detect if the original request was indeed via HTTPS. Luckily,
most reverse proxies set the `x-forwarded-proto` header flag with the original request scheme.

#### Azure

Azure has a slightly different way of signaling encrypted connections.
It uses `x-arr-ssl` header as a flag to mark https trafic.

## Examples

Those are full example apps using Koa SSLify to enforce HTTPS.

### Without Reverse Proxy

This example starts 2 servers for app.

- First HTTP server is listening on port 8080 and redirects to second one
- Second HTTPS server is listening on port 8081

```javascript
const Koa = require('koa');
const http = require('http');
const https = require('https');
const fs = require('fs');
const { default: enforceHttps } = require('koa-sslify');

const app = new Koa();

// Force HTTPS using default resolver
app.use(enforceHttps({
  port: 8081
}));

// index page
app.use(ctx => {
  ctx.body = "hello world from " + ctx.request.url;
});

// SSL options
var options = {
  key: fs.readFileSync('server.key'),
  cert: fs.readFileSync('server.crt')
}

// start the server
http.createServer(app.callback()).listen(8080);
https.createServer(options, app.callback()).listen(8081);
```

### With Reverse Proxy

This exmple starts single http server which is designed to run behind
reverse proxy like Heroku.

```javascript
const Koa = require('koa');
const {
  default: enforceHttps,
  xForwardedProtoResolver: resolver
} = require('koa-sslify');

var app = new Koa();

// Force HTTPS via x-forwarded-proto compatible resolver
app.use(enforceHttps({ resolver }));

// index page
app.use((ctx) => {
  ctx = "hello world from " + ctx.request.url;
});

// proxy will bind this port to it's 443 and 80 ports
app.listen(3000);
```

## Advanced Redirect Setting

### Redirect Methods

By default only `GET` and `HEAD` methods are whitelisted for redirect.
koa-sslify will respond with `403` (`405` if `specCompliantDisallow` option is set) on all other methods.
You can change whitelisted methods by passing `redirectMethods` array to options.

### Internal Redirect Support \[POST/PUT\]

**By default there is no HTTP(S) methods whitelisted for `307 internal redirect`.**
You can define custom whitelist of methods for `307` by passing `internalRedirectMethods` array to options.
This should be useful if you want to support `POST` and `PUT` delegation from `HTTP` to `HTTPS`.
For more info see [this](http://www.checkupdown.com/status/E307.html) article.

### Skip Default Port in Redirect URL

**By default this plugin exclude port from redirect url if it's set to `443`.**
Since `443` is default port for `HTTPS` browser will use it by default anyway so there
is no need to explicitly return it as part of URL. Anyway in case you need to **always return port as part of URL string**
you can pass options with `skipDefaultPort: false` to do the trick.

*Thanks to [@MathRobin](https://github.com/MathRobin) for implementation of this as well as port skipping itself. Thanks to [@sethb0](https://github.com/sethb0) for specCompliantDisallow feature and implementation.*

## License
MIT

## Credits
This project is heavily inspired by [Florian Heinemann's](https://github.com/florianheinemann) [express-sslify](https://github.com/florianheinemann/express-sslify)
and [Vitaly Domnikov's](https://github.com/dotcypress) [koa-force-ssl](https://github.com/dotcypress/koa-force-ssl).
