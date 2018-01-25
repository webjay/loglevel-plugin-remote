# loglevel-plugin-remote

A [loglevel](https://github.com/pimterry/loglevel) plugin for sending  browser logs to a server.

[![NPM version](https://img.shields.io/npm/v/loglevel-plugin-remote.svg?style=flat-square)](https://www.npmjs.com/package/loglevel-plugin-remote)[![Build Status](https://img.shields.io/travis/kutuluk/loglevel-plugin-remote/master.svg?style=flat-square)](https://travis-ci.org/kutuluk/loglevel-plugin-remote)

## Features

- Sends logs asynchronously with a specified frequency using only one request at a time.
- Slows the frequency of sending after a fail and restores after success.
- Supports Bearer authentication scheme.
- Provides light string interpolation like console and node.js (%s, %d, %j, %o).
- Provides the ability to add any values ​​to the logs and specify a custom format.
- Single file, no dependencies, weighs in at 2.3KB minified and gzipped.

## Installation

```sh
npm i loglevel-plugin-remote --save
```

## API

**This plugin is under active development and should be considered as an unstable. No guarantees regarding API stability are made. Backward compatibility is guaranteed only by path releases.**

#### ```apply(loglevel, options)```
This method applies the plugin to the loglevel.

#### Parameters
```loglevel``` - the root logger, imported from loglevel package

```options``` - an optional configuration object

```javascript
const defaults = {
  url: '/logger',
  headers: {},
  token: '',
  onUnauthorized: (failedToken) => {},
  timeout: 0,
  interval: 1000,
  backoff: (interval) => {
    const multiplier = 2;
    const jitter = 0.1;
    const limit = 30000;
    let next = interval * multiplier;
    if (next > limit) next = limit;
    next += next * jitter * Math.random();
    return next;
  },
  capacity: 500,
  stacktrace: {
    levels: ['trace', 'warn', 'error'],
    depth: 3,
    excess: 0,
  },
  timestamp: () => new Date().toISOString(),
  format: remote.plain,
};
```

* **url** (string) - a URL of the server logging API

* **headers** (object) - this is to support custom headers passed down to the post request. For example { 'my-header': 'myHeaderValue' }

* **token** (string) - a token for Bearer authentication scheme (see [RFC 6750](https://tools.ietf.org/html/rfc6750)), e.g. [UUID](https://en.wikipedia.org/wiki/Universally_unique_identifier) or [JWT](https://jwt.io/). By default is `''`. If you set the value to `undefined`, the sending of logs will be suspended until the `setToken` method is called, but the logs will still accumulate in the queue.

* **onUnauthorized** (function) - this function will be called after the sending of logs is suspended due to an authentication error. A token that has not been authenticated will be passed to the function.

* **timeout** (number) - a timeout in milliseconds (see [XMLHttpRequest.timeout](https://developer.mozilla.org/docs/Web/API/XMLHttpRequest/timeout))

* **interval** (number) - a time in milliseconds between sending messages. By default is 1000 (one second).

* **backoff** (function) - a function used to increase the sending interval after each failed send. By default, it doubles the interval and adds 10% jitter. Having reached the value of 30 seconds, the interval increase stops. After successful sending, the interval will be reset to the initial value.

* **capacity** (number) - the size of the queue in which messages are accumulated between sending. By default is 500. Overflow will delete the oldest messages. It is forbidden to set the value to 0 - in this case the default value will be used.

* **stacktrace** (object) - object for configuring a stacktrace with the following properties:
  * **levels** (array) - lots of levels for which to add the stack trace. By default is `trace`, `warn` and `error`.
  * **depth** (number) - number of stack trace lines. By default is 3.
  * **excess** (number) - number of excess lines generated by following plugins (each plugin adds one line). This number of lines will be removed from the top of the stack trace. By default is 0.

* **timestamp** (function) - a function that returns a timestamp. By default, it returns the time in the ISO format (see [ISO 8601](https://en.wikipedia.org/wiki/ISO_8601))

* **format** (function) - this option defines the logs format. This function will generate a log, obtaining as a single argument a log object that looks like this:

```javascript
const log = {
  message: 'Text',
  level: {
    label: 'info',
    value: 2,
  },
  logger: 'child',
  timestamp: '2017-05-29T12:53:46.000Z',
  stacktrace: '',
};
```

When the function returns a string, the logs will be sent as plain text. The default value is function `remote.plain`:

```javascript
function plain(log) {
  return `[${log.timestamp}] ${log.level.label.toUpperCase()}${log.logger ? ` (${log.logger})` : ''}: ${log.message}${log.stacktrace ? `\n${log.stacktrace}` : ''}`;
}
```

Then when you call:

```javascript
log.getLogger('child').info('Info message')
log.error('Error message')
```
the logs look like this:

```
[2017-05-29T12:53:46.000Z] INFO (child): Info message
[2017-05-29T12:53:46.001Z] ERROR: Error message
    at http://localhost/test.js:12:5
```

When the function returns an object, the logs will be sent in json format. You can use the preset function `remote.json`:

```javascript
function json(log) {
  log.level = log.level.label;
  return log;
}
```

then the logs will look like this:

```json
{
    "logs": [
        {
            "message": "Info message",
            "level": "info",
            "logger": "child",
            "timestamp": "2017-05-29T12:53:46.000Z",
            "stacktrace": ""
        },
        {
            "message": "Error message",
            "level": "error",
            "logger": "",
            "timestamp": "2017-05-29T12:53:46.001Z",
            "stacktrace": "    at http://localhost/test.js:12:5"
        }
    ]
}
```

You can include any values ​​in the log. For example:

```javascript
import log from 'loglevel';
import remote from 'loglevel-plugin-remote';

const getCounter = () => {
  let count = 1;
  return () => count++;
};
const counter = getCounter();

const customPlain = log => `[${counter()}] ${log.message}`;

/*
const customJSON = log => ({
  msg: log.message,
  count: counter(),
});
*/

remote.apply(log, { format: customPlain });
// remote.apply(log, { format: customJSON });

log.enableAll();
log.info('Message one');
log.info('Message two');

```

customPlain:

```
[1] Message one
[2] Message two
```

customJSON:

```json
{
    "logs": [
        {
            "msg": "Message one",
            "count": 1
        },
        {
            "msg": "Message two",
            "count": 2
        }
    ]
}
```

#### ```setToken(token)```

This method only has an effect after a successful call to the apply method and is used to assign a new authentication token. If you pass the value `undefined`, the sending of logs will be suspended until the next `setToken` call, but the logs will still accumulate in the queue.

#### ```disable()```
This method cancels the effect of the plugin.

## Base usage

### Browser directly

```html
<script src="https://unpkg.com/loglevel/dist/loglevel.min.js"></script>
<script src="https://unpkg.com/loglevel-plugin-remote@^0.6/dist/loglevel-plugin-remote.min.js"></script>

<script>
  var logger = log.noConflict();
  var sender = remote.noConflict();
  sender.apply(logger);
  logger.warn('message');
</script>
```

### ES6
```javascript
import log from 'loglevel';
import remote from 'loglevel-plugin-remote';

remote.apply(log);
log.warn('message');
```

### CommonJS
```javascript
var log = require('loglevel');
var remote = require('loglevel-plugin-remote');

remote.apply(log);
log.warn('message');
```

### AMD
```javascript
define(['loglevel', 'loglevel-plugin-remote'], function(log, remote) {
  remote.apply(log);
  log.warn('message');
});
```

## Example

Code
```javascript
var log = require('loglevel');
var remote = require('loglevel-plugin-remote');

log.enableAll();

remote.apply(log);

log.info('Log levels:');
log.trace('trace message');
log.debug('debug message');
log.info('info message');
log.warn('warn message');
log.error('error message');
```

Output in a log server
```
[2017-05-29T12:53:46.000Z] INFO: Log levels:
[2017-05-29T12:53:46.000Z] TRACE: trace message
    at http://localhost/js/test.js:9:5
[2017-05-29T12:53:46.000Z] DEBUG: debug message
[2017-05-29T12:53:46.000Z] INFO: info message
[2017-05-29T12:53:46.000Z] WARN: warn message
    at http://localhost/js/test.js:12:5
[2017-05-29T12:53:46.000Z] ERROR: error message
    at http://localhost/js/test.js:13:5
```

Code
```javascript
log.info('String interpolation: %% %t %s', 'one', 'two');
log.info('Number interpolation: %d %d %d %d', 16, 1e6, '16', '1e6');
```

Output in a log server
```
[2017-05-29T12:53:46.000Z] INFO: String interpolation: % %t one two
[2017-05-29T12:53:46.000Z] INFO: Number interpolation: 16 1000000 16 1000000
```

Code
```javascript
log.info('Object interpolation:');

function Rectangle(width, height) {
  this.width = width;
  this.height = height;
}
var object = new Rectangle(10, 10);
log.info('%s, %d, %o, %j', object, object, object, object, object);

var date = new Date();
log.info('date: %o', date);

var error = new Error('My error');
log.info('error: %o', error);

var string = 'My string';
log.info('string: %o', string);

var number = 123;
log.info('number: %o', number);

var bool = true;
log.info('boolean: %o', bool);

var array = [1, 2, 3];
log.info('array: %o', array);
```

Output in a log server
```
[2017-05-29T12:53:46.000Z] INFO: Object interpolation:
[2017-05-29T12:53:46.000Z] INFO: [object Object], NaN, Rectangle{"height":10,"width":10}, {"height":10,"width":10} [object Object]
[2017-05-29T12:53:46.000Z] INFO: date: Date<"2017-06-04T13:16:01.455Z">
[2017-05-29T12:53:46.000Z] INFO: error: Error{}
[2017-05-29T12:53:46.000Z] INFO: string: String<"My string">
[2017-05-29T12:53:46.000Z] INFO: number: Number<123>
[2017-05-29T12:53:46.000Z] INFO: boolean: Boolean<true>
[2017-05-29T12:53:46.000Z] INFO: array: Array[1,2,3]
```

## Multiple plugins

Code
```javascript
var log = require('loglevel');
var remote = require('loglevel-plugin-remote');
var mock = require('loglevel-plugin-mock');

// To clean the loglevel-plugin-mock line in the stack trace:
// options = { stacktrace: { excess: 1 } }
remote.apply(log, { stacktrace: { excess: 1 } });
mock.apply(log);

var array = [1, 2, 3];
log.warn('array: %o', array);
```

Output in a log server
```
[2017-05-29T12:53:46.000Z] WARN: array: Array[1,2,3]
    at http://localhost/js/test.js:11:5
```
