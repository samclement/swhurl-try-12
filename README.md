# swhurl-try-12

throwaway: team platform T2 Kotlin caller, delete me

A Kotlin ([Micronaut](https://micronaut.io/)) app on the [swhurl platform](https://github.com/samclement/swhurl-platform), made from the [swhurl Kotlin template](https://github.com/samclement/swhurl-app-template-kotlin). Every push to `main` is checked, published to `ghcr.io/<owner>/swhurl-try-12` and deployed to staging; production changes through **Promote to production**. What the image must provide, the checks and the dependency updates: the [template's guide](https://github.com/samclement/swhurl-app-template-kotlin#readme). What the app needs from the platform: [`swhurl.yaml`](swhurl.yaml).

An HTTP service: `HttpController.kt` answers on port 8080; the platform probes `GET /healthz`.

## Callers and calling other apps

`Platform.kt` holds what the platform gives the app ([calling another app](https://github.com/samclement/swhurl-platform/blob/main/docs/apps.md#calling-another-app)):

- **Who is calling.** `CallerFilter` (in `HttpController.kt`) runs before every route except `/healthz` and puts a `Caller` on the request: type `app` with the app's name (it sent a token the platform issued for this app alone), `user` with a signed-in person's email, or `anonymous`. A bad token is answered 401. Every request's log line and span carry `caller.type` and `caller.id`. For a route only apps may use, call `platform.requireApp(caller)`.
- **Calling another app.** Add `calls: [orders]` to `swhurl.yaml` before the app is created (afterwards: `make app-calls` on the platform, or the app page's **Calls** form). Then `platform.call("orders", "/path")` reaches the same environment's `orders` with this app's token for it; the callee's spans join the same trace. `GET /call/orders` in `HttpController.kt` is an example to replace.

## Notifications

`platform.notify("Order backlog", message = "412 orders waiting", priority = "high")` (in `Platform.kt`) sends a message to the notification channel of the team that owns this app ([notifications](https://github.com/samclement/swhurl-platform/blob/main/docs/apps.md#notifications)). Nothing is declared in `swhurl.yaml`: the platform gives every app the address and a token. The message arrives titled `<app>/<env>: Order backlog`. The call returns whether the platform accepted it and never throws; an app may send 30 an hour, and a team without a channel delivers nothing. `POST /notify` in `HttpController.kt` is an example to replace. Locally there is no notify service: `notify` returns `false` and sends nothing.

## Events

`Events.kt` publishes and receives events on the platform's NATS ([events](https://github.com/samclement/swhurl-platform/blob/main/docs/apps.md#events)). Declare them in `swhurl.yaml` before the app is created (afterwards: `make app-events` on the platform):

```yaml
events:
  publish: [created]               # other apps see swhurl-try-12.created
  subscribe: [orders.created]      # <app>.<event>
```

Then `events.publish("created", mapOf("id" to 42))` and `events.subscribe("orders.created") { event -> ... }`. An event is a CloudEvents envelope (`event.data` is what was published); a handler runs inside the publisher's trace, and if it throws the event comes back, five tries in all. `subscribe` keeps trying until it is receiving, since a new app can start before NATS knows it. `HttpController.kt` publishes `visited` on every request to `/`: an example to replace. Locally there is no NATS: `publish` returns `null` and `subscribe` never calls back.

### The events this app offers: `asyncapi.yaml`

[`src/main/resources/asyncapi.yaml`](src/main/resources/asyncapi.yaml) describes each event swhurl-try-12 publishes ([AsyncAPI 3](https://www.asyncapi.com/docs)): its name, what it means, the schema of its payload and an example. It is the declaration. The platform reads it to show other apps what they can subscribe to, and `events.publish()` throws `EventInvalid` for an event that is not in it or a payload that does not fit, locally as well as on the platform.

To add an event, add to `asyncapi.yaml` a channel (its `address` is the event's name), a message with a `payload` schema and an example, and a `send` operation; then call `events.publish("<name>", data)`. `AsyncApiTest` checks that every event has a description, a schema and an example that fits it. Nothing else is declared: after the push, the platform reads the file for the new image and other apps can subscribe ([what an app offers](https://github.com/samclement/swhurl-platform/blob/main/docs/apps.md#what-an-app-offers)).

## The API this app offers: `openapi.yaml`

[Micronaut OpenAPI](https://micronaut-projects.github.io/micronaut-openapi/latest/guide/) writes the API description from the controllers when the app is built ([OpenAPI 3](https://spec.openapis.org/oas/v3.0.1)): every route, with the types it takes and returns, and the summary in its `@Operation`. [`src/main/resources/openapi.yaml`](src/main/resources/openapi.yaml) is that description, committed; the app serves it at `/openapi.yaml`, and the platform reads the file to show other apps what they can call. `@Hidden` keeps a route out of it (`/healthz`).

After changing a route, run `./gradlew openapi` and commit `openapi.yaml`: `OpenApiTest` fails while the file and the routes disagree, so a change to the API always shows in a pull request.

## Without the platform

Locally there are no platform keys or tokens: requests without a token work as before and a request with one is refused; to call another app, set `SWHURL_SERVICE_<NAME>_URL` to a local stub.

## Local development

Needs a JDK 25 (Gradle downloads everything else):

```bash
./gradlew run                           # the agent stays off locally (the image loads it)
curl http://localhost:8080/healthz
./gradlew check                         # compile and tests, as CI does
```
