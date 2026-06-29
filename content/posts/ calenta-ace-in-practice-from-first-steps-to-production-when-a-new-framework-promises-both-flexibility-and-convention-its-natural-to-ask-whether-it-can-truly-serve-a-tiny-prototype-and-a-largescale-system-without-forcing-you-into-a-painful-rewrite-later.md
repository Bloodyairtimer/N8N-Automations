# Calenta Ace in Practice: From First Steps to Production

When a new framework promises both flexibility and convention, it’s natural to ask whether it can truly serve a tiny prototype and a large‑scale system without forcing you into a painful rewrite later. After spending a few weeks with Calenta Ace on internal tooling and a couple of customer migrations, I’ve found that its design strikes a balance that many teams appreciate: a solid core that stays out of your way, a plugin system that lets you add only what you need, and enough built‑in tooling to keep the development loop tight. Below is a walk‑through of how the pieces fit together, what to watch out for, and how you can start using it today.

## 1. Core Concepts: What the Framework Actually Provides

At the heart of Calenta Ace is a lightweight runtime that wires together three main abstractions:

* **Service Container** – a simple dependency‑injection registry that resolves objects by type or name. You register factories, singletons, or scoped instances once, and the container handles lifetime management for you.
* **Middleware Pipeline** – inspired by the classic request‑response chain, each middleware receives a context, can mutate it, and decides whether to pass control downstream. This makes cross‑cutting concerns like logging, authentication, or request validation feel like ordinary code rather than framework magic.
* **Plugin Architecture** – plugins are just packages that expose a `register(container)` function. They can add services, middleware, configuration defaults, or even CLI commands. Because the registration step is explicit, you always know what a plugin contributes, and you can replace or remove it without digging through hidden conventions.

Together, these pieces give you a *framework‑like* experience without locking you into a monolithic structure. You start with a barebones project, add the plugins you need, and let the container wire everything up automatically.

## 2. Getting a Project Off the Ground

The official CLI (`calenta`) scaffolds a minimal layout that follows a convention‑over‑configuration approach. Running `calenta init my-service` creates:

```
my-service/
 ├─ src/
 │   ├─ index.ts          # entry point
 │   └─ plugins/          # place for local plugins
 ├─ config/
 │   └─ default.ts        # typed configuration schema
 └─ calenta.json          # CLI metadata
```

The generated `index.ts` looks like this:

```ts
import { createApp } from '@calenta/ace';
import { logger } from '@calenta/plugin-logger';
import { jsonBody } from '@calenta/plugin-bodyparser';

const app = createApp();

// plug in middleware
app.use(logger());
app.use(jsonBody());

// define a simple route
app.get('/hello', ctx => {
  ctx.body = { message: 'Hello world' };
});

app.listen(3000);
```

Notice there’s no boilerplate for setting up a server, configuring routes, or wiring dependencies. The `createApp` call builds a service container, loads any plugins listed in `calenta.json`, and returns an object that exposes the middleware pipeline and routing helpers. The result is a runnable HTTP service in under ten lines of code.

If you already have a Node.js Express application, migration is largely a matter of replacing the Express `app` with the Calenta Ace equivalent and moving your route handlers into the new middleware style. Because the middleware signature (`(ctx, next) => Promise<void>`) is deliberately similar to Express, many handlers can be copied verbatim, with only minor adjustments to how you access request/response data (via `ctx.request` and `ctx.response`).

## 3. Leveraging Plugins for Real‑World Concerns

One of the framework’s strengths is its ecosystem of first‑party plugins that solve common backend tasks without you having to reinvent the wheel.

* **Authentication** – `@calenta/plugin-auth` provides JWT and OAuth2 strategies. Registration adds a `authenticate()` middleware that populates `ctx.user` on successful validation.
* **Observability** – `@calenta/plugin-tracing` hooks into the middleware pipeline to emit OpenTelemetry spans, while `@calenta/plugin-metrics` exports Prometheus counters for request latency and error rates.
* **Caching** – `@calenta/plugin-cache` offers a thin wrapper around Redis or in‑memory stores, exposing a `getOrSet(key, fn)` helper that you can call from any service or middleware.

Because each plugin registers its own services, you can swap implementations easily. For example, during local development you might use the in‑memory cache, then switch to the Redis-backed version in staging by simply changing the plugin configuration in `calenta.json`:

```json
{
  "plugins": [
    "@calenta/plugin-logger",
    "@calenta/plugin-bodyparser",
    { "name": "@calenta/plugin-cache", "options": { "provider": "redis", "url": "redis://cache:6379" } }
  ]
}
```

The framework reads this file at startup, injects the specified options, and makes the cache service available via the container.

## 4. Performance Tuning and Pitfalls

Even with a thoughtful default setup, there are a few patterns that can unintentionally hurt performance or maintainability.

### Over‑Customizing Before Learning the Basics
It’s tempting to replace the default middleware pipeline with a custom chain the moment you see a slight inconvenience. Doing so obscures the flow of data and makes debugging harder. Instead, start with the built‑in logger and body parser, observe the request lifecycle in your logs, and only add custom middleware when you have a clear cross‑cutting concern that the existing plugins don’t address.

### Ignoring the Official Plugin Ecosystem
Because the core is intentionally minimal, some teams reach for hand‑rolled solutions for tasks like validation or rate limiting. The official plugins are already integrated with the container and middleware system, meaning they benefit from automatic lifecycle management and typed configuration. Re‑implementing them often leads to duplicated effort and subtle bugs around error handling.

### Skipping Project Structure Conventions
The CLI enforces a layout that separates concerns: configuration lives in `config/`, local plugins in `src/plugins/`, and entry points in `src/`. Deviating from this layout can confuse the CLI’s automatic discovery (e.g., it won’t load a plugin placed outside the registered folder). Stick to the convention unless you have a compelling reason to reorganize, and then update `calenta.json` accordingly.

### Treating the Framework as a Monolith
Although you can build a large application in a single service, Calenta Ace shines when you split responsibilities into independently deployable microservices. Each service gets its own container, its own set of plugins, and its own configuration profile. This isolation makes it easier to scale specific components (like a high‑throughput ingest service) without over‑provisioning others.

A practical performance win comes from enabling the built‑in caching plugin for expensive data‑fetching middleware. By wrapping a database call in `cache.getOrSet('user:' + id, () => db.findUser(id))`, you reduce round‑trips dramatically for read‑heavy endpoints, and the plugin handles cache invalidation through its exposed `del` method.

## 5. A Small Case Study: Moving a Legacy API to the Cloud

A recent project involved migrating a legacy Node.js REST API that relied on ad‑hoc middleware and a global configuration singleton. The goals were:

1. Add structured logging and request tracing.
2. Introduce strong typing for request payloads.
3. Deploy the service to Kubernetes with horizontal autoscaling.

Using Calenta Ace, we took the following steps:

* Created a new service with `calenta init legacy-api`.
* Migrated each Express route to a Calenta Ace handler, preserving the business logic almost unchanged.
* Added `@calenta/plugin-logger` (JSON format) and `@calenta/plugin-tracing` (OpenTelemetry) to the plugin list.
* Defined a Zod schema for each endpoint and used `@calenta/plugin-validator` to automatically reject malformed bodies before they hit the handler.
* Replaced the global config singleton with a typed configuration file (`config/default.ts`) that the container injects into any service that requests it.
* Added a Kubernetes manifest that references the Dockerfile produced by `calenta build`. The manifest sets resource requests based on observed CPU usage from the built‑in metrics plugin.

After deployment, we observed a 30 % reduction in latency for the 95th percentile request, largely because the tracing plugin highlighted a blocking synchronous file read that we moved to an async stream. The autoscaler responded to the metrics exported by the plugin, adding pods during traffic spikes without manual intervention.

## Conclusion

Calenta Ace isn’t trying to be everything to everyone; it aims to give you a predictable foundation that you can extend deliberately. By keeping the core small, exposing a clear plugin contract, and providing sensible defaults for configuration, logging, and dependency management, it lets teams focus on domain code rather than framework plumbing. If you’re evaluating a new backend stack, give the CLI a spin, try a plugin or two, and see how the middleware pipeline feels in your own service. The investment in learning the conventions pays off quickly when you need to scale, maintain, or split your services later on.