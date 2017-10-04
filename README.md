# loglevel-plugin-remote
Plugin for sending [loglevel](https://github.com/pimterry/loglevel) messages to a remote log server

# Features
- Sends asynchronously and does not slow down the application
- Messages are sent one by one, so the order is maintained
- Support any server that accepts a Post request
- Support string substitutions like console and node.js (%s, %d, %j, %o)

## Installation

```sh
npm install loglevel-plugin-remote --save
```

## API

**This plugin is under active development and should be considered as an unstable. No guarantees regarding API stability are made. Backward compatibility is guaranteed only by path releases.**

```javascript
apply(log, options);
```

**log** - root logger, imported from loglevel package

**options** - configuration object

```javascript
const defaults = {
  url: '/logger',
  token: '',
  timeout: 0,
  interval: 100,
  backoff: (interval) => {
    const multiplier = 2;
    const jitter = 0.1;
    const limit = 30000;
    let next = interval * multiplier;
    if (next > limit) next = limit;
    next += next * jitter * Math.random();
    return next;
  },
  capacity: 0,
  trace: ['trace', 'warn', 'error'],
  depth: 0,
  json: false,
  timestamp: () => new Date().toISOString(),
};
```

- **url** - log server URL
- **token** - token for Bearer authentication scheme (e.g. UUID: "9fda563b-7103-4f3c-ad93-4fe0085ce75c") (see [RFC 6750](https://tools.ietf.org/html/rfc6750))
- **timeout** - timeout in milliseconds (see [XMLHttpRequest.timeout](https://developer.mozilla.org/docs/Web/API/XMLHttpRequest/timeout))
- **interval** - time in milliseconds between sending messages
- **backoff** - a function used to increase the sending interval after each failed send. By default, it doubles the interval and adds 10% jitter. Having reached the value of 30 seconds, the interval increase stops. After successful sending, the interval will be reset to the initial value.
- **capacity** - the size of the queue in which messages are accumulated between sending. Overflow will delete the oldest messages. By default is 0, which makes the queue unlimited.
- **trace** - lots of levels for which to add the stack trace
- **depth** - the number of following plugins (affects the number of rows to clear the stack trace)
- **json** - when set to true, it sends messages in json format:

```json
{
    "messages": [
        {
            "message": "Message one",
            "stacktrace": "at http://localhost/test.js:11:5",
            "timestamp": "2017-05-29T12:53:46.000Z",
            "level": "info",
            "logger": ""
        },
        {
            "message": "Message two",
            "stacktrace": "at http://localhost/test.js:12:5",
            "timestamp": "2017-05-29T12:53:46.001Z",
            "level": "warn",
            "logger": ""
        }
    ]
}
```

- **timestamp** - a function that returns a timestamp and used when messages sending in json format. By default, it returns the time in the ISO format (see [ISO 8601](https://en.wikipedia.org/wiki/ISO_8601))

## Base usage

### Browser directly

```html
<script src="https://unpkg.com/loglevel/dist/loglevel.min.js"></script>
<script src="https://unpkg.com/loglevel-plugin-remote/dist/loglevel-plugin-remote.min.js"></script>

<script>
  var logger = log.noConflict();
  remote.noConflict().apply(logger);
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

log.setLevel('trace');

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
Log levels:
trace message
    at http://localhost/js/test.js:9:5
debug message
info message
warn message
    at http://localhost/js/test.js:12:5
error message
    at http://localhost/js/test.js:13:5
```

Code
```javascript
log.info('String substitutions: %% %t %s', 'one', 'two');
log.info('Number substitutions: %d %d %d %d', 16, 1e6, '16', '1e6');
```

Output in a log server
```
String substitutions: % %t one two
Number substitutions: 16 1000000 16 1000000
```

Code
```javascript
log.info('Object substitutions:');

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
Object substitutions:
[object Object], NaN, Rectangle{"height":10,"width":10}, {"height":10,"width":10} [object Object]
date: Date<"2017-06-04T13:16:01.455Z">
error: Error{}
string: String<"My string">
number: Number<123>
boolean: Boolean<true>
array: Array[1,2,3]
```

## Multiple plugins

Code
```javascript
var log = require('loglevel');
var remote = require('loglevel-plugin-remote');
var prefix = require('loglevel-plugin-prefix');

// To clean the loglevel-plugin-prefix line in the stack-trace:
// options = { depth: 1 }
remote.apply(log, { depth: 1 });
prefix.apply(log);

var array = [1, 2, 3];
log.warn('array: %o', array);
```

Output in a log server
```
[12:53:46] WARN: array: Array[1,2,3]
    at http://localhost/js/test.js:11:5
```
