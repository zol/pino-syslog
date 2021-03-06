# pino-syslog

**Lead maintainer:** [jsumners](https://github.com/jsumners)

*pino-syslog* is a so called "transport" for the [pino][pino] logger. *pino-syslog* receives *pino* logs from `stdin`
and transforms them into [RFC3164][rfc3164] or [RFC5424][rfc5424] (syslog) formatted messages which are written to
`stdout`. The default output format is RFC5424.

This transport **does not** send messages to a remote, or even local, syslog compatible server. It merely reformats the
logs into syslog compatible strings. To send logs to a syslog server, use the [pino-socket][pino-socket] transport.
For example:

```bash
$ node your-app.js | pino-syslog | pino-socket -a syslog.example.com
```

[pino]: https://www.npmjs.com/package/pino
[rfc3164]: https://tools.ietf.org/html/rfc3164
[rfc5424]: https://tools.ietf.org/html/rfc5424
[pino-socket]: https://www.npmjs.com/package/pino-socket

## RFC3164

This RFC mandates that the maximum number of bytes that a syslog message may be
is `1024`. Thus, *pino-syslog* will do one of two things when this limit is exceeded:

1. Output a JSON error log, with syslog header, that includes the original log's `time` and `level` properties, a
  `originalSize` property set to the number of bytes the original log message consumed, and a `msg` property set to
  "message exceeded syslog 1024 byte limit".
2. Truncate the message to fit within the 1024 byte limit when the `messageOnly` configuration option is set to `true`.

This means you *can* lose data if your log messages are too large. If that is to be the case, you should investigate
the `includeProperties` option to reduce your log size. But, really, you should investigate what it is you are logging.

## RFC5424

This RFC does not limit the message size except to say that the ***receiver*** may impose a maximum. Thus, *pino-syslog*
does not impose a length limit when conforming to this RFC. There are a couple of things to note, though:

1. We do not currently support the structured data portion of the log header. This section of each log is always `-`.
2. If the data to be logged includes `req.id` then it will be used as the message id portion of the log. For example,
  the data `{req: {id: '1234'}}` would have '1234' as the message id in the resulting formatted log.

These caveats may be configurable in a later version.

## Example

Given the log:

```json
{"pid":94473,"hostname":"MacBook-Pro-3","level":30,"msg":"hello world","time":1459529098958,"v":1}
```

*pino-syslog* will write out:

```
<134>1 2016-04-01T16:44:58Z MacBook-Pro-3 - 94473 - - {"pid":94473,"hostname":"MacBook-Pro-3","level":30,"msg":"hello world","time":1459529098958,"v":1}
```

Or, in RFC3164 mode:

```
<134>Apr  1 16:44:58 MacBook-Pro-3 none[94473]: {"pid":94473,"hostname":"MacBook-Pro-3","level":30,"msg":"hello world","time":1459529098958,"v":1}
```

Putting it all together:

```bash
$ echo '{"pid":94473,"hostname":"MacBook-Pro-3","level":30,"msg":"hello world","time":1459529098958,"v":1}' | node pino-syslog                                                       [s:0 l:8025]
<134>1 2016-04-01T16:44:58Z MacBook-Pro-3 - 94473 - - {"pid":94473,"hostname":"MacBook-Pro-3","level":30,"msg":"hello world","time":1459529098958,"v":1}
```

## Install

You should install *pino-syslog* globally so that it can be used as a utility:

```bash
$ npm install --production -g pino-syslog
```

## Command Line Arguments

There is only one argument available: `--config` (`-c`). This argument is used to load a JSON configuration file.

### Configuration

*pino-syslog* supports configuration via a JSON file. The default options are:

```json
{
  "modern": true,
  "appname": "none",
  "cee": false,
  "facility": 16,
  "includeProperties": [],
  "messageOnly": false,
  "tz": "Etc/UTC",
  "newline": false,
  "structuredData": "-"
}
```

+ `modern` (boolean): indicates if RFC5424 (`true`) or RFC3164 (`false`) should be used.
+ `appname` (string): sets the name of the application in the 'TAG' portion of the syslog header.
+ `cee` (boolean): denotes whether or not to prefix the message field with `@cee: `. This will only work if
  `messageOnly` is `false`.
+ `facility` (number): a valid [facility number][facility], `[0 - 23]`.
+ `includeProperties` (array<string>): a list of property names from the original *pino* log to include in the formatted
  message. This is only applicable if `messageOnly` is `false`.
+ `messageOnly` (boolean): indicates if the message field should contain only the `msg` property of the *pino* log, or
  if it should be stringified JSON.
+ `tz` (string): any [valid timezone string][tzstring] that [moment][moment] will recognize. The timestamp field of the
  syslog header will be sent according to this setting.
+ `newline` (boolean): terminate with a newline
+ `structuredData` (string): [structured data](https://tools.ietf.org/html/rfc5424#section-6.3) to send with an RFC5424 message.

[facility]: https://tools.ietf.org/html/rfc3164#section-4.1.1
[tzstring]: https://en.wikipedia.org/wiki/List_of_tz_database_time_zones
[moment]: http://momentjs.com/timezone/docs/#/using-timezones/parsing-in-zone/

## License

[MIT License](http://jsumners.mit-license.org/)
