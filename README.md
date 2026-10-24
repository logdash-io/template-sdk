# Logdash SDK Contribution Guide

Thank you for your interest in contributing to Logdash by creating a new SDK — we truly appreciate your support!

Below you'll find a step-by-step guide to help you get started.

## ✅ Code ownership and approvals

All SDK repositories are hosted under the Logdash GitHub organization.

As a contributor and maintainer, you will have full access to develop, maintain, and improve the SDK. However, any changes to the main branch must be reviewed and approved by the Logdash team.

This policy ensures consistency, security, and quality across all SDKs.

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
- Writing tests, which must at least check that:
  - log and metric payloads carry `app` when it is set and leave the key out when it is not.
  - `track` sends `POST {host}/events` with the `project-api-key` and `content-type: application/json` headers, and a body with `name`, `props` and a UUID v4 `id`.
  - event bodies carry `app` the same way as logs and metrics.
  - event bodies carry `userId` when it is passed.
  - an event with an invalid name, or a `userId` that contains `@`, is not sent.
  - flush sends queued events.
- Creating SDK documentation that follows the format of our existing SDKs:
  - [JavaScript SDK](https://github.com/logdash-io/js-sdk)
  - [Python SDK](https://github.com/logdash-io/python-sdk)
  - You can copy the README and adjust the code sections as needed.
  - Add an "Events" section after metrics, as described in [Documentation](#documentation).
- Writing build and deployment instructions.
- Creating a CI/CD pipeline for automatic deployment to the relevant package manager.

## 🔧 Creating a Logdash Instance

Our SDKs expose a way to initialize a Logdash instance:

```
const { metrics, logger, track } = createLogdash({
  apiKey: 'your-api-key',
  app: 'api',
  host: '',
  verbose: false
});
```

This may vary depending on the target language and ecosystem.

### Configuration Parameters

| Parameter | Required | Default          | Description                                                                                    |
| --------- | -------- | ---------------- | ---------------------------------------------------------------------------------------------- |
| `apiKey`  | No       | -                | API key for authentication. If omitted, logs and events are printed to the local console only. |
| `app`     | No       | -                | Name of this app, for example `api` or `desktop`. Sent with every log, metric and event.       |
| `host`    | No       | `api.logdash.io` | Custom API host. Useful when using self-hosted instances.                                      |
| `verbose` | No       | `false`          | Enables verbose/debug logging in the SDK itself.                                               |

The domain key, passed as `apiKey`, is shared by all apps of a domain (for example a desktop app, an API and a landing page), and `app` names this one.
Older service keys ignore `app`.
Namespaced or child loggers inherit it.

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
- `app` (optional): The `app` option. Include the key only when `app` is set; otherwise leave it out entirely (no `null`, no empty string).
- `sequenceNumber` (optional): An incrementing local counter to help preserve log order within the same millisecond.

**Note:** This field is especially useful when multiple logs are created in the same moment but delivered out of order. We batch logs every 100ms on the backend and use `sequenceNumber` to determine proper order.

If your SDK sends logs in batches with `POST /logs/batch`, every entry of the `logs` array carries `app` the same way.

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

As with logs, the body carries `app` only when it is set.

## 🎯 Events

### Available Methods

```
track(name, props?, options?)
```

- `name`: The event name, for example `agent_run`.
- `props` (optional): A flat object of properties that describe the event.
- `options` (optional): Carries `userId`, the signed-in user's internal ID.

Use your language's idiomatic form of this signature, for example a `user_id` keyword argument in Python.

### Async Behavior

`track` should also be **non-blocking**: like logs, it sends events in the background and never throws into the application.
Flush and graceful shutdown should send queued events too.

### Behavior

`track` should send each event in its own request to this endpoint:

> `POST {host}/events`

The request carries the headers `project-api-key: <apiKey>` and `content-type: application/json`.

Payload must include:

- `name`: The event name, matching `^[a-z][a-z0-9_]{0,63}$`.
  `pageview`, `pageleave` and `browser_error` belong to the website tracker, and the API rejects them.
- `props` (optional): A flat object of up to 10 properties.
  Keys match `^[a-z][a-z0-9_]{0,39}$`, and values are strings, numbers or booleans.
  The API stores values as strings cut to 100 characters, and drops invalid entries and values that contain `@`, never the event.
- `userId` (optional): The signed-in user's internal ID, a string or a number of up to 256 characters, never an email address.
  A value that contains `@` gets `400`.
  Send it as is: the API stores only `sha256(siteId + ':' + userId)`, the same hash the website tracker's `identify` sends, so a user is one person across the website and every app.
- `id`: A UUID v4 generated for each event.
  Retries of the event reuse it, so the event is stored once.
- `app` (optional): The `app` option, sent exactly as with logs and metrics.
  With a domain key it names the app, and the API uses `default` when it is absent.
  A service key uses its service name.

Include each optional key only when it is set; otherwise leave it out entirely (no `null`, no empty string).

The API responds with `202` when it accepts the event, `400` for an invalid body, `401` for an unknown API key and `429` when the hourly limit is reached.

Without `apiKey`, `track` does what logs do in that mode: it prints the event to the local console only.

### Validation

Before sending or printing an event, check `name` against `^[a-z][a-z0-9_]{0,63}$`.
An invalid name, or a `userId` that contains `@`, drops the event with a warning, never an exception.

### Documentation

Add an "Events" section after metrics to your SDK's README.
It shows one example, in your language, that tracks `agent_run` with props and a `userId`:

```
track('agent_run', { agent: 'support', steps: 4, success: true }, { userId: user.id });
```

Then it says, in plain sentences:

- Events show up in the domain's Analytics as goals, with a daily trend.
- Pass `userId` for signed-in users, so one person counts once across the website and apps.
- Without it, Logdash counts a visitor of that app that changes every day.
- Never put personal data in props.

---

If you have any questions or need support, don’t hesitate to contact us at [contact@logdash.io](mailto:contact@logdash.io). Happy coding! 🚀
