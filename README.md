# Template for creating new SDK

If you are there, you probably want to contribute to logdash and create a new SDK. Thank you for that :)

Steps:

- reach out to us - contact@logdash.io,
- we as logdash org owners create public repository inside logdash org and assign you as a maintainer,
- we as logdash org owners create account in choosen package manager (npm, pypi, nuget, etc...) and give you a token,

What needs to be done within SDK (you don't have to do every step):

- you write SDK code,
- you provide SDK documentation compliant with general readme. See https://github.com/logdash-io/js-sdk or https://github.com/logdash-io/python-sdk.
  Just copy the readme and adjust parts with code to match your SDK
- you write an instruction how to build it and deploy it,
- you create an automatic pipeline which auto deploys the package.

## Creating logdash instance

In general we want to spawn logdash using more or less something like

```
const { metrics, logger } = createLogdash({
  apiKey: 'your-api-key'
  host: '',
  verbose: ''
})
```

Obviously this may vary due to different technologies.

These are parameters we use

| Parameter | Required | Default        | Description                                                                                                              |
| --------- | -------- | -------------- | ------------------------------------------------------------------------------------------------------------------------ |
| `apiKey`  | no       | -              | Api key used to authorize against logdash servers. If you don't provide one, logs will be logged into local console only |
| `host`    | no       | api.logdash.io | Custom API host, useful with self-hosted instances                                                                       |
| `verbose` | no       | false          | Useful for debugging purposes                                                                                            |

## Logger

### Methods

Logger should expose such methods

```
error()
warn()
info()
log()
http()
verbose()
debug()
silly()
```

### Being async

Logger should not pause code execution. It should just execute in the "background".

### Colors

Logger should log into the console. Use these colors

```
const LOG_LEVEL_COLORS: Record<LogLevel, [number, number, number]> = {
	[LogLevel.ERROR]: [231, 0, 11],
	[LogLevel.WARN]: [254, 154, 0],
	[LogLevel.INFO]: [21, 93, 252],
	[LogLevel.HTTP]: [0, 166, 166],
	[LogLevel.VERBOSE]: [0, 166, 0],
	[LogLevel.DEBUG]: [0, 166, 62],
	[LogLevel.SILLY]: [80, 80, 80],
};
```

### What it should actually do

Calling any logger method should call this endpoint https://api.logdash.io/docs#/Logs/LogCoreController_create.

- `createdAt` - should be set to `now()` in ISO format,
- `message` - is the raw log message,
- `level` - self explanatory,
- `sequenceNumber` - you should have some sort of offline counter which increments itself by one every time you add new log. This field
  is optional and entire API can work without it. It is here for cases when you add two logs in the exact same millisecond but log #2 arrives
  first to the server. The smallest possible time unit by which we can order logs is millisecond so to order logs created in the same moment
  we are batching them and "indexing" based on `sequenceNumber` each 100ms on the backend.

### Parameters parsing

#### Multiple params

Javascript sdk supports any number of parameters, so I can call

```
logger.log("Hello","from","logdash")
```

and it will work. If possible, make your sdk behave the same.

#### Stringifying params

Javascript sdk supports passing raw objects as params and will do its best to do `JSON.stringify(param)` whenever it is possible. If your language
let's you pass objects to params, please add serialization as well.

## Metrics

### Methods

Metrics should expose such methods

```
set(metricName, value)
mutate(metricName, value)
```

### Being async

Same as logs - this should be completely non-blocking operration.

### What it should actually do

It should call this endpoint https://api.logdash.io/docs#/Metrics/MetricCoreController_recordMetric
