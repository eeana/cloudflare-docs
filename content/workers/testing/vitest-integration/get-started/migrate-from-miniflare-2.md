---
title: Migrate from Miniflare 2's test environments
weight: 2
pcx_content_type: concept
meta:
  description: Migrate from [Miniflare 2](https://github.com/cloudflare/miniflare?tab=readme-ov-file) to the Workers Vitest integration.
---

# Migrate from Miniflare 2's test environments

[Miniflare 2](https://github.com/cloudflare/miniflare?tab=readme-ov-file) provided custom environments for Jest and Vitest in the `jest-environment-miniflare` and `vitest-environment-miniflare` packages respectively.
The `@cloudflare/vitest-pool-workers` package provides similar functionality using modern Miniflare versions and the [`workerd` runtime](https://github.com/cloudflare/workerd). `workerd` is the same JavaScript/WebAssembly runtime that powers Cloudflare Workers. Using `workerd` practically eliminates behavior mismatches between your tests and deployed code. Refer to the [Miniflare 3 announcement](https://blog.cloudflare.com/miniflare-and-workerd) for more information.

{{<Aside type="warning">}}

Cloudflare no longer provides a Jest testing environment for Workers. If you previously used Jest, you will need to [migrate to Vitest](https://vitest.dev/guide/migration.html#migrating-from-jest) first, then follow the rest of this guide. Vitest provides built-in support for TypeScript, ES modules, and hot-module reloading for tests out-of-the-box.

{{</Aside>}}

{{<Aside type="warning">}}

The Workers Vitest integration does not support testing Workers using the service worker format. [Migrate to ES modules format](/workers/reference/migrate-to-module-workers/) first.

{{</Aside>}}

## Install the Workers Vitest integration

First, you will need to uninstall the old environment and install the new pool. Vitest environments can only customize the global scope, whereas pools can run tests using a completely different runtime. In this case, the pool runs your tests inside [`workerd`](https://github.com/cloudflare/workerd) instead of Node.js.

```sh
$ npm uninstall vitest-environment-miniflare
$ npm install --save-dev --save-exact vitest@1.5.0
$ npm install --save-dev @cloudflare/vitest-pool-workers
```

## Update your Vitest configuration file

After installing the Workers Vitest configuration, update your Vitest configuration file to use the pool instead. Most Miniflare configuration previously specified `environmentOptions` can be moved to `poolOptions.workers.miniflare` instead. Refer to [Miniflare's `WorkerOptions` interface](https://github.com/cloudflare/workers-sdk/blob/main/packages/miniflare/README.md#interface-workeroptions) for supported options and the [Miniflare version 2 to 3 migration guide](https://miniflare.dev/get-started/migrating#api-changes) for more information. If you relied on configuration stored in a `wrangler.toml` file, set `wrangler.configPath` too.

```diff
---
filename: vitest.config.js
---
+ import { defineWorkersConfig } from "@cloudflare/vitest-pool-workers/config";

  export default defineWorkersConfig({
    test: {
-     environment: "miniflare",
-     environmentOptions: { ... },
+     poolOptions: {
+       workers: {
+         miniflare: { ... },
+         wrangler: { configPath: "./wrangler.toml" },
+       },
+     },
    },
  });
```

## Update your TypeScript configuration file

If you are using TypeScript, update your `tsconfig.json` to include the correct ambient `types`:

```diff
---
filename: tsconfig.json
---
  {
    "compilerOptions": {
      ...,
      "types": [
        "@cloudflare/workers-types/experimental"
-       "vitest-environment-miniflare/globals"
+       "@cloudflare/vitest-pool-workers"
      ]
    },
  }
```

## Access bindings

To access [bindings](/workers/runtime-apis/bindings/) in your tests, use the `env` helper from the `cloudflare:test` module.

```diff
---
filename: index.spec.js
---
  import { it } from "vitest";
+ import { env } from "cloudflare:test";

  it("does something", () => {
-   const env = getMiniflareBindings();
    // ...
  });
```

If you are using TypeScript, add an ambient `.d.ts` declaration file defining a `ProvidedEnv` `interface` in the `cloudflare:test` module to control the type of `env`:

```ts
---
filename: env.d.ts
---
declare module "cloudflare:test" {
  interface ProvidedEnv {
    NAMESPACE: KVNamespace;
  }
  // ...or if you have an existing `Env` type...
  interface ProvidedEnv extends Env {}
}
```

## Use isolated storage

Isolated storage is now enabled by default. You no longer need to include `setupMiniflareIsolatedStorage()` in your tests.

```diff
---
filename: index.spec.js
---
- const describe = setupMiniflareIsolatedStorage();
+ import { describe } from "vitest";
```

## Work with `waitUntil()`

The `new ExecutionContext()` constructor and `getMiniflareWaitUntil()` function are now `createExecutionContext()` and `waitOnExecutionContext()` respectively. Note `waitOnExecutionContext()` now returns an empty `Promise<void>` instead of a `Promise` resolving to the results of all `waitUntil()`ed `Promise`s.

```diff
---
filename: index.spec.js
---
+ import { createExecutionContext, waitOnExecutionContext } from "cloudflare:test";

  it("does something", () => {
    // ...
-   const ctx = new ExecutionContext();
+   const ctx = createExecutionContext();
    const response = worker.fetch(request, env, ctx);
-   await getMiniflareWaitUntil(ctx);
+   await waitOnExecutionContext(ctx);
  });
```

## Mock outbound requests

The `getMiniflareFetchMock()` function has been replaced with the new `fetchMock` helper from the `cloudflare:test` module. `fetchMock` has the same type as the return type of `getMiniflareFetchMock()`. There are a couple of differences between `fetchMock` and the previous return value of `getMiniflareFetchMock()`:

- `fetchMock` is deactivated by default, whereas previously it would start activated. This deactivation prevents unnecessary buffering of request bodies if you are not using `fetchMock`. You will need to call `fetchMock.activate()` before calling `fetch()` to enable it.
- `fetchMock` is reset at the start of each test run, whereas previously, interceptors added in previous runs would apply to the current one. This ensures test runs are not affected by previous runs.

```diff
---
filename: index.spec.js
---
  import { beforeAll, afterAll } from "vitest";
+ import { fetchMock } from "cloudflare:test";

- const fetchMock = getMiniflareFetchMock();
  beforeAll(() => {
+   fetchMock.activate();
    fetchMock.disableNetConnect();
    fetchMock
      .get("https://example.com")
      .intercept({ path: "/" })
      .reply(200, "data");
  });
  afterAll(() => fetchMock.assertNoPendingInterceptors());
```

## Use Durable Object helpers

The `getMiniflareDurableObjectStorage()`, `getMiniflareDurableObjectState()`, `getMiniflareDurableObjectInstance()`, and `runWithMiniflareDurableObjectGates()` functions have all been replaced with a single `runInDurableObject()` function from the `cloudflare:test` module. The `runInDurableObject()` function accepts a `DurableObjectStub` with a callback accepting the Durable Object instance and corresponding `DurableObjectState` as arguments. Consolidating these functions into a single function simplifies the API surface, and ensures instances are accessed with the correct request context and [gating behavior](https://blog.cloudflare.com/durable-objects-easy-fast-correct-choose-three/). Refer to the [Test APIs page](/workers/testing/vitest-integration/test-apis/) for more details.

```diff
---
filename: index.spec.js
---
+ import { env, runInDurableObject } from "cloudflare:test";

  it("does something", async () => {
-   const env = getMiniflareBindings();
    const id = env.OBJECT.newUniqueId();
+   const stub = env.OBJECT.get(id);

-   const storage = await getMiniflareDurableObjectStorage(id);
-   doSomethingWith(storage);
+   await runInDurableObject(stub, async (instance, state) => {
+     doSomethingWith(state.storage);
+   });

-   const state = await getMiniflareDurableObjectState(id);
-   doSomethingWith(state);
+   await runInDurableObject(stub, async (instance, state) => {
+     doSomethingWith(state);
+   });

-   const instance = await getMiniflareDurableObjectInstance(id);
-   await runWithMiniflareDurableObjectGates(state, async () => {
-     doSomethingWith(instance);
-   });
+   await runInDurableObject(stub, async (instance) => {
+     doSomethingWith(instance);
+   });
  });
```

The `flushMiniflareDurableObjectAlarms()` function has been replaced with the `runDurableObjectAlarm()` function from the `cloudflare:test` module. The `runDurableObjectAlarm()`  function accepts a single `DurableObjectStub` and returns a `Promise` that resolves to `true` if an alarm was scheduled and the `alarm()` handler was executed, or `false` otherwise. To "flush" multiple instances' alarms, call `runDurableObjectAlarm()` in a loop.

```diff
---
filename: index.spec.js
---
+ import { env, runDurableObjectAlarm } from "cloudflare:test";

  it("does something", async () => {
-   const env = getMiniflareBindings();
    const id = env.OBJECT.newUniqueId();
-   await flushMiniflareDurableObjectAlarms([id]);
+   const stub = env.OBJECT.get(id);
+   const ran = await runDurableObjectAlarm(stub);
  });
```

Finally, the `getMiniflareDurableObjectIds()` function has been replaced with the `listDurableObjectIds()` function from the `cloudflare:test` module. The `listDurableObjectIds()` function now accepts a `DurableObjectNamespace` instance instead of a namespace `string` to provide stricter typing. Note the `listDurableObjectIds()` function now respects isolated storage. If enabled, IDs of objects created in other tests will not be returned.

```diff
---
filename: index.spec.js
---
+ import { env, listDurableObjectIds } from "cloudflare:test";

  it("does something", async () => {
-   const ids = await getMiniflareDurableObjectIds("OBJECT");
+   const ids = await listDurableObjectIds(env.OBJECT);
  });
```