# Logdash SDK Contribution Guide

Thank you for your interest in contributing to Logdash by creating a new SDK — we truly appreciate your support!

Below you'll find a step-by-step guide to help you get started.

## 📬 Initial Setup

To begin:

1. **Reach out to us** at [contact@logdash.io](mailto:contact@logdash.io).
2. The Logdash team will:
   - Create a public repository under the [Logdash GitHub organization](https://github.com/logdash-io).
   - Assign you as a maintainer of that repository.
   - Create an account on the relevant package registry (e.g., npm, PyPI, NuGet) and provide you with an access token.

## 📦 SDK Responsibilities

As an SDK maintainer, your responsibilities may include:

- Writing the core SDK code.
- Creating SDK documentation that follows the format of our existing SDKs:
  - [JavaScript SDK](https://github.com/logdash-io/js-sdk)
  - [Python SDK](https://github.com/logdash-io/python-sdk)
  - You can copy the README and adjust the code sections as needed.
- Writing build and deployment instructions.
- Creating a CI/CD pipeline for automatic deployment to the relevant package manager.

## 🔧 Creating a Logdash Instance

Our SDKs expose a way to initialize a Logdash instance:

```
const { metrics, logger } = createLogdash({
  apiKey: 'your-api-key',
  host: '',
  verbose: false
});
```

This may vary depending on the target language and ecosystem.

### Configuration Parameters

| Parameter | Required | Default          | Description                                                                         |
| --------- | -------- | ---------------- | ----------------------------------------------------------------------------------- |
| `apiKey`  | No       | -                | API key for authentication. If omitted, logs are printed to the local console only. |
| `host`    | No       | `api.logdash.io` | Custom API host. Useful when using self-hosted instances.                           |
| `verbose` | No       | `false`          | Enables verbose/debug logging in the SDK itself.                                    |

---

## 📝 Logger

### Available Methods

Your logger implementation should expose the following methods:

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

---

### Async Behavior

Logging must be **non-blocking** — it should never pause or delay application execution. All operations should run in the background.

---

### Console Output with Colors

Use the following RGB color scheme when outputting to the console:

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

---

### Behavior

Each log method should send a request to this endpoint:

> `https://api.logdash.io/docs#/Logs/LogCoreController_create`

Payload must include:

- `createdAt`: Current timestamp in ISO format.
- `message`: Raw log message.
- `level`: Log level.
- `sequenceNumber` (optional): An incrementing local counter to help preserve log order within the same millisecond.

**Note:** This field is especially useful when multiple logs are created in the same moment but delivered out of order. We batch logs every 100ms on the backend and use `sequenceNumber` to determine proper order.

---

### Parameter Handling

#### Multiple Parameters

The JavaScript SDK allows multiple parameters, e.g.:

```
logger.log("Hello", "from", "Logdash");
```

If possible, your SDK should support similar behavior.

#### Object Serialization

JavaScript SDK automatically serializes objects using `JSON.stringify`. If your language supports it, please implement a similar mechanism to handle non-string parameters.

## 📈 Metrics

### Available Methods

```
set(metricName, value)
mutate(metricName, value)
```

- `set`: Assigns an absolute value to a metric.
- `mutate`: Modifies the metric by a given delta (positive or negative).

### Async Behavior

Metric operations should also be **non-blocking**.

### Behavior

Each metric method should send data to this endpoint:

> `https://api.logdash.io/docs#/Metrics/MetricCoreController_recordMetric`

---

If you have any questions or need support, don’t hesitate to contact us at [contact@logdash.io](mailto:contact@logdash.io). Happy coding! 🚀
