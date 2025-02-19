---
title: Single Fetch
---

# Single Fetch

<docs-warning>This is an unstable API and will continue to change, do not adopt in production</docs-warning>

Single fetch is a new data loading strategy and streaming format. When you enable Single Fetch, Remix will make a single HTTP call to your server on client-side transitions, instead of multiple HTTP calls in parallel (one per loader). Additionally, Single Fetch also allows you to send down naked objects from your `loader` and `action`, such as `Date`, `Error`, `Promise`, `RegExp`, and more.

## Overview

Remix introduced support for "Single Fetch" ([RFC][rfc]) behind the [`future.unstable_singleFetch`][future-flags] flag in [`v2.9.0`][2.9.0] which allows you to opt-into this behavior. Single Fetch will be the default in [React Router v7][merging-remix-and-rr].

Enabling Single Fetch is intended to be low-effort up-front, and then allow you to adopt all breaking changes iteratively over time. You can start by applying the minimal required changes to [enable Single Fetch][start], then use the [migration guide][migration-guide] to make incremental changes in your application to ensure a smooth, non-breaking upgrade to [React Router v7][merging-remix-and-rr].

Please also review the [Breaking Changes][breaking-changes] so you can be aware of some of the underlying behavior changes, specifically around serialization and status/header behavior.

## Enabling Single Fetch

**1. Enable the future flag**

```ts filename=vite.config.ts lines=[6]
export default defineConfig({
  plugins: [
    remix({
      future: {
        // ...
        unstable_singleFetch: true,
      },
    }),
    // ...
  ],
});
```

**2. Deprecated `fetch` polyfill**

Single Fetch requires using [`undici`][undici] as your `fetch` polyfill, or using the built-in `fetch` on Node 20+, because it relies on APIs available there that are not in the `@remix-run/web-fetch` polyfill. Please refer to the [Undici][undici-polyfill] section in the 2.9.0 release notes below for more details.

- If you are using Node 20+, remove any calls to `installGlobals()` and use Node's built-in `fetch` (this is the same thing as `undici`).

- If you are managing your own server and calling `installGlobals()`, you will need to call `installGlobals({ nativeFetch: true })` to use `undici`.

  ```diff
  - installGlobals();
  + installGlobals({ nativeFetch: true });
  ```

- If you are using `remix-serve`, it will use `undici` automatically if Single Fetch is enabled.

- If you are using miniflare/cloudflare worker with your remix project, ensure your [compatibility flag][compatibility-flag] is set to `2023-03-01` or later as well.

**3. Remove document-level `headers` implementation (if you have one)**

The [`headers`][headers] export is not longer used when single fetch is enabled. In many cases you may have been just re-returning the headers from your loader `Response` instances to apply them to document requests, and if so, you may can likely just remove the export and those Repsonse headers will apply to document requests automatically. If you were doing more complex logic for document headers in the `headers` function, then you will need to migrate those to the new [Response Stub][responsestub] instance in your `loader` functions.

**4. Add `nonce` to `<RemixServer>` (if you are using a CSP)**

The `<RemixServer>` component renders inline scripts that handle the streaming data on the client side. If you have a [content security policy for scripts][csp] with [nonce-sources][csp-nonce], you can use `<RemixServer nonce>` to pass through the nonce to these `<script>` tags.

**5. Replace `renderToString` (if you are using it)**

For most Remix apps it's unlikely you're using `renderToString`, but if you have opted into using it in your `entry.server.tsx`, then continue reading, otherwise you can skip this step.

In order to maintain consistency between document and data requests, `turbo-stream` is also used as the format for sending down data in initial document requests. This means that once opted-into Single Fetch, your application can no longer use [`renderToString`][rendertostring] and must use a React streaming renderer API such as [`renderToPipeableStream`][rendertopipeablestream] or [`renderToReadableStream`][rendertoreadablestream]) in [`entry.server.tsx`][entry-server].

This does not mean you _have_ to stream down your HTTP response, you can still send the full document at once by leveraging the `onAllReady` option in `renderToPipeableStream`, or the `allReady` promise in `renderToReadableStream`.

On the client side, this also means that your need to wrap your client-side [`hydrateRoot`][hydrateroot] call in a [`startTransition`][starttransition] call because the streamed data will be coming down wrapped in a `Suspense` boundary.

## Breaking Changes

There are a handful of breaking changes introduced with Single Fetch - some of which you need to handle up-front when you enable the flag, and some you can handle incrementally after enabling the flag. You will need to ensure all of these have been handled prior to updating to the next major version.

**Changes that need to be addressed up front:**

- **Deprecated `fetch` polyfill**: The old `installGlobals()` polyfill doesn't work for Single Fetch, you must either use the native Node 20 `fetch` API or call `installGlobals({ nativeFetch: true })` in your custom server to get the [undici-based polyfill][undici-polyfill]
- **Deprecated `headers` export**: The [`headers`][headers] function is no longer used when Single Fetch is enabled, in favor of the new `response` stub passed to your `loader`/`action` functions

**Changes to be aware of that you may need to handle over-time:**

- **[New streaming Data format][streaming-format]**: Single fetch uses a new streaming format under the hood via [`turbo-stream`][turbo-stream], which means that we can stream down more complex data than just JSON
- **No more auto-serialization**: Naked objects returned from `loader` and `action` functions are no longer automatically converted into a JSON `Response` and are serialized as-is over the wire
- **Updates to type inference**: To get the most accurate type inference, you should do two things:
  - Add `@remix-run/react/future/single-fetch.d.ts` to the end of your `tsconfig.json`'s `compilerOptions.types` array
  - Begin using `unstable_defineLoader`/`unstable_defineAction` in your routes
    - This can be done incrementally - you should have _mostly_ accurate type inference in your current state
- [**Opt-in `action` revalidation**][action-revalidation]: Revalidation after an `action` `4xx`/`5xx` `Response` is now opt-in, versus opt-out

## Adding a New Route with Single Fetch

With Single Fetch enabled, you can go ahead and author routes that take advantage of the more powerful streaming format and [`response` stub][responsestub].

<docs-info>In order to get proper type inference, you first need to add `@remix-run/react/future/single-fetch.d.ts` to the end of your `tsconfig.json`'s `compilerOptions.types` array. You can read more about this in the [Type Inference section][type-inference-section].</docs-info>

With Single Fetch you can return the following data types from your loader: `BigInt`, `Date`, `Error`, `Map`, `Promise`, `RegExp`, `Set`, `Symbol`, and `URL`.

```tsx
// routes/blog.$slug.tsx
import { unstable_defineLoader as defineLoader } from "@remix-run/node";

export const loader = defineLoader(
  async ({ params, response }) => {
    const { slug } = params;

    const comments = fetchComments(slug);
    const blogData = await fetchBlogData(slug);

    response.headers.set("Cache-Control", "max-age=300");

    return {
      content: blogData.content, // <- string
      published: blogData.date, // <- Date
      comments, // <- Promise
    };
  }
);

export default function BlogPost() {
  const blogData = useLoaderData<typeof loader>();
  //    ^? { content: string, published: Date, comments: Promise }

  return (
    <>
      <Header published={blogData.date} />
      <BlogContent content={blogData.content} />
      <Suspense fallback={<CommentsSkeleton />}>
        <Await resolve={blogData.comments}>
          {(comments) => (
            <BlogComments comments={comments} />
          )}
        </Await>
      </Suspense>
    </>
  );
}
```

## Migrating a Route with Single Fetch

If you are currently returning `Response` instances from your loaders (i.e., `json`/`defer`) then you shouldn't _need_ to make many changes to your app code to take advantage of Single Fetch.

However, to better prepare your upgrade to [React Router v7][merging-remix-and-rr] in the future, we recommend that you start making the following changes on a route-by-route basis, as that is the easiest way to validate that updating the headers and data types doesn't break anything.

### Type Inference

Without Single Fetch, any plain Javascript object returned from a `loader` or `action` is automatically serialized into a JSON response (as if you returned it via `json`). The type inference assumes this is the case and infers naked object returns as if they were JSON serialized.

With Single Fetch, naked objects will be streamed directly, so the built-in type inference is no longer accurate once you have opted-into Single Fetch. For example, they would assume that a `Date` would be serialized to a string on the client 😕.

In order to ensure you get the proper types when using Single Fetch, we've included a set of type overrides that you can include in your `tsconfig.json`'s `compilerOptions.types` array which aligns the types with the Single Fetch behavior:

```json
{
  "compilerOptions": {
    //...
    "types": [
      // ...
      "@remix-run/react/future/single-fetch.d.ts"
    ]
  }
}
```

🚨 Make sure the single-fetch types come after any other Remix packages in `types` so that they override those existing types.

#### Loader/Action Definition Utilities

To enhance type-safety when defining loaders and actions with Single Fetch, you can use the new `unstable_defineLoader` and `unstable_defineAction` utilities:

```ts
import { unstable_defineLoader as defineLoader } from "@remix-run/node";

export const loader = defineLoader(({ request }) => {
  //                                  ^? Request
});
```

Not only does this give you types for arguments (and deprecates `LoaderFunctionArgs`), but it also ensures you are returning single-fetch compatible types:

```ts
export const loader = defineLoader(() => {
  return { hello: "world", badData: () => 1 };
  //                       ^^^^^^^ Type error: `badData` is not serializable
});

export const action = defineAction(() => {
  return { hello: "world", badData: new CustomType() };
  //                       ^^^^^^^ Type error: `badData` is not serializable
});
```

Single-fetch supports the following return types:

```ts
type Serializable =
  | undefined
  | null
  | boolean
  | string
  | symbol
  | number
  | bigint
  | Date
  | URL
  | RegExp
  | Error
  | Array<Serializable>
  | { [key: PropertyKey]: Serializable } // objects with serializable values
  | Map<Serializable, Serializable>
  | Set<Serializable>
  | Promise<Serializable>;
```

There are also client-side equivalents un `defineClientLoader`/`defineClientAction` that don't have the same return value restrictions because data returned from `clientLoader`/`clientAction` does not need to be serialized over the wire:

```ts
import { unstable_defineLoader as defineLoader } from "@remix-run/node";
import { unstable_defineClientLoader as defineClientLoader } from "@remix-run/react";

export const loader = defineLoader(() => {
  return { msg: "Hello!", date: new Date() };
});

export const clientLoader = defineClientLoader(
  async ({ serverLoader }) => {
    const data = await serverLoader<typeof loader>();
    //    ^? { msg: string, date: Date }
    return {
      ...data,
      client: "World!",
    };
  }
);

export default function Component() {
  const data = useLoaderData<typeof clientLoader>();
  //    ^? { msg: string, date: Date, client: string }
}
```

<docs-info>These utilities are primarily for type inference on `useLoaderData` and its equivalents. If you have a resource route that returns a `Response` and is not consumed by Remix APIs (such as `useFetcher`), then you can just stick with your normal `loader`/`action` definitions. Converting those routes to use `defineLoader`/`defineAction` would cause type errors because `turbo-stream` cannot serialize a `Response` instance.</docs-info>

#### `useLoaderData`, `useActionData`, `useRouteLoaderData`, `useFetcher`

These methods do not require any code changes on your part - adding the Single Fetch types will cause their generics to deserialize correctly:

```ts
export const loader = defineLoader(async () => {
  const data = await fetchSomeData();
  return {
    message: data.message, // <- string
    date: data.date, // <- Date
  };
});

export default function Component() {
  // ❌ Before Single Fetch, types were serialized via JSON.stringify
  const data = useLoaderData<typeof loader>();
  //    ^? { message: string, date: string }

  // ✅ With Single Fetch, types are serialized via turbo-stream
  const data = useLoaderData<typeof loader>();
  //    ^? { message: string, date: Date }
}
```

#### `useMatches`

`useMatches` requires a manual cast to specify the loader type in order to get proper type inference on a given `match.data`. When using Single Fetch, you will need to replace the `UIMatch` type with `UIMatch_SingleFetch`:

```diff
  let matches = useMatches();
- let rootMatch = matches[0] as UIMatch<typeof loader>;
+ let rootMatch = matches[0] as UIMatch_SingleFetch<typeof loader>;
```

#### `meta` Function

`meta` functions also require a generic to indicate the current and ancestor route loader types in order to properly type the `data` and `matches` parameters. When using Single Fetch, you will need to replace the `MetaArgs` type with `MetaArgs_SingleFetch`:

```diff
  export function meta({
    data,
    matches,
- }: MetaArgs<typeof loader, { root: typeof rootLoader }>) {
+ }: MetaArgs_SingleFetch<typeof loader, { root: typeof rootLoader }>) {
    // ...
  }
```

### Headers

The [`headers`][headers] function is no longer used when Single Fetch is enabled.
Instead, your `loader`/`action` functions now receive a mutable `ResponseStub` unique to that execution:

- To alter the status of your HTTP Response, set the `status` field directly:
  - `response.status = 201`
- To set the headers on your HTTP Response, use the standard [`Headers`][mdn-headers] APIs:
  - `response.headers.set(name, value)`
  - `response.headers.append(name, value)`
  - `response.headers.delete(name)`

```ts
export const action = defineAction(
  async ({ request, response }) => {
    if (!loggedIn(request)) {
      response.status = 401;
      response.headers.append("Set-Cookie", "foo=bar");
      return { message: "Invalid Submission!" };
    }
    await addItemToDb(request);
    return null;
  }
);
```

You can also throw these response stubs to short circuit the flow of your loaders and actions:

```tsx
export const loader = defineLoader(
  ({ request, response }) => {
    if (shouldRedirectToHome(request)) {
      response.status = 302;
      response.headers.set("Location", "/");
      throw response;
    }
    // ...
  }
);
```

Each `loader`/`action` receives its own unique `response` instance so you cannot see what other `loader`/`action` functions have set (which would be subject to race conditions). The resulting HTTP Response status and headers are determined as follows:

- Status Code
  - If all status codes are unset or have values <300, the deepest status code will be used for the HTTP response
  - If any status codes are set to a value >=300, the shallowest >=300 value will be used for the HTTP Response
- Headers
  - Remix tracks header operations and will replay them on a fresh `Headers` instance after all handlers have completed
  - These are replayed in order - action first (if present) followed by loaders in top-down order
  - `headers.set` on any child handler will overwrite values from parent handlers
  - `headers.append` can be used to set the same header from both a parent and child handler
  - `headers.delete` can be used to delete a value set by a parent handler, but not a value set from a child handler

Because Single Fetch supports naked object returns, and you no longer need to return a `Response` instance to set status/headers, the `json`/`redirect`/`redirectDocument`/`defer` utilities should be considered deprecated when using Single Fetch. These will remain for the duration of v2 so you don't need to remove them immediately. They will likely be removed in the next major version, so we recommend remove them incrementally between now and then.

These utilities will remain for the rest of Remix v2, and it's likely that in a future version they'll be available via something like [`remix-utils`][remix-utils] (or they're also very easy to re-implement yourself).

For v2, you may still continue returning normal `Response` instances and they'll apply status codes in the same way as the `response` stub, and will apply all headers via `headers.set` - overwriting any same-named header values from parents. If you need to append a header, you will need to switch from returning a `Response` instance to using the new `response` parameter.

To ensure you can adopt these features incrementally, our goal is that you can enable Single Fetch without changing all of your `loader`/`action` functions to leverage the `response` stub. Then over time, you can incrementally convert individual routes to leverage the new `response` stub.

### Client Loaders

If your app has routes using [`clientLoader`][client-loader] functions, it's important to note that the behavior of Single Fetch will change slightly. Because `clientLoader` is intended to give you a way to opt-out of calling the server `loader` function - it would be incorrect for the Single Fetch call to execute that server loader. But we run all loaders in parallel and we don't want to _wait_ to make the call until we know which `clientLoader`'s are actually asking for server data.

For example, consider the following `/a/b/c` routes:

```ts
// routes/a.tsx
export function loader() {
  return { data: "A" };
}

// routes/a.b.tsx
export function loader() {
  return { data: "B" };
}

// routes/a.b.c.tsx
export function loader() {
  return { data: "C" };
}

export function clientLoader({ serverLoader }) {
  await doSomeStuff();
  const data = await serverLoader();
  return { data };
}
```

If a user navigates from `/ -> /a/b/c`, then we need to run the server loaders for `a` and `b`, and the `clientLoader` for `c` - which may eventually (or may not) call it's own server `loader`. We can't decide to include the `c` server `loader` in a single fetch call when we want to fetch the `a`/`b` `loader`'s, nor can we delay until `c` actually makes the `serverLoader` call (or returns) without introducing a waterfall.

Therefore, when you export a `clientLoader` that route opts-out of Single Fetch and when you call `serverLoader` it will make a single fetch to get only it's route server `loader`. All routes that do not export a `clientLoader` will be fetched in a singular HTTP request.

So, on the above route setup a navigation from `/ -> /a/b/c` will result in a singular single-fetch call up front for routes `a` and `b`:

```
GET /a/b/c.data?_routes=routes/a,routes/b
```

And then when `c` calls `serverLoader`, it'll make it's own call for just the `c` server `loader`:

```
GET /a/b/c.data?_routes=routes/c
```

### Resource Routes

Because of the new [streaming format][streaming-format] used by Single Fetch, raw JavaScript objects returned from `loader` and `action` functions are no longer automatically converted to `Response` instances via the `json()` utility. Instead, in navigational data loads they're combined with the other loader data and streamed down in a `turbo-stream` response.

This poses an interesting conundrum for [resource routes][resource-routes] which are unique because they're intended to be hit individually -- and not always via Remix APIs. They can also be accessed via any other HTTP client (`fetch`, `cURL`, etc.).

If a resource route is intended for consumption by internal Remix APIs, we _want_ to be able to leverage the `turbo-stream` encoding to unlock the ability to stream down more complex structures such as `Date` and `Promise` instances. However, when accessed externally, we'd probably prefer to return the more easily consumable JSON structure. Thus, the behavior is slightly ambiguous if you return a raw object in v2 - should it be serialized via `turbo-stream` or `json()`?

To ease backwards-compatibility and ease the adoption of the Single Fetch future flag, Remix v2 will handle this based on whether it's accessed from a Remix API or externally. In the future Remix will require you to return your own [JSON response][returning-response] if you do not want raw objects to be streamed down for external consumption.

The Remix v2 behavior with Single Fetch enabled is as follows:

- When accessing from a Remix API such as `useFetcher`, raw Javascript objects will be returned as `turbo-stream` responses, just like normal loaders and actions (this is because `useFetcher` will append the `.data` suffix to the request)

- When accessing from an external tool such as `fetch` or `cURL`, we will continue this automatic conversion to `json()` for backwards-compatibility in v2:

  - Remix will log a deprecation warning when this situation is encountered
  - At your convenience, you can update impacted resource route handlers to return a `Response` object
  - Addressing these deprecation warnings will better prepare you for the eventual Remix v3 upgrade

  ```tsx filename=app/routes/resource.tsx bad
  export function loader() {
    return {
      message: "My externally-accessed resource route",
    };
  }
  ```

  ```tsx filename=app/routes/resource.tsx good
  export function loader() {
    return Response.json({
      message: "My externally-accessed resource route",
    });
  }
  ```

Note: It is _not_ recommended to use `defineLoader`/`defineAction` for externally-accessed resource routes that need to return specific `Response` instances. It's best to just stick with `loader`/`LoaderFunctionArgs` for these cases.

#### Response Stub and Resource Routes

As discussed above, the `headers` export is deprecated in favor of a new [`response` stub][responsestub] passed to your `loader` and `action` functions. This is somewhat confusing in resource routes, though, because you get to return the _actual_ `Response` - there's no real need for a "stub" concept because there's no merging results from multiple loaders into a single Response:

```tsx filename=app/routes/resource.tsx
// Using your own Response is the most straightforward approach
export async function loader() {
  const data = await getData();
  return Response.json(data, {
    status: 200,
    headers: {
      "X-Custom": "whatever",
    },
  });
}
```

To keep things consistent, resource route `loader`/`action` functions will still receive a `response` stub and you can use it if you need to (maybe to share code amongst non-resource route handlers):

```tsx filename=app/routes/resource.tsx
// But you can still set values on the response stub
export async function loader({
  response,
}: LoaderFunctionArgs) {
  const data = await getData();
  response?.status = 200;
  response?.headers.set("X-Custom", "whatever");
  return Response.json(data);
}
```

It's best to try to avoid using the `response` stub _and also_ returning a `Response` with custom status/headers, but if you do, the following logic will apply":

- The `Response` instance status will take priority over any `response` stub status
- Headers operations on the `response` stub `headers` will be re-played on the returned `Response` headers instance

## Additional Details

### Streaming Data Format

Previously, Remix used `JSON.stringify` to serialize your loader/action data over the wire, and needed to implement a custom streaming format to support `defer` responses.

With Single Fetch, Remix now uses [`turbo-stream`][turbo-stream] under the hood which provides first class support for streaming and allows you to automatically serialize/deserialize more complex data than JSON. The following data types can be streamed down directly via `turbo-stream`: `BigInt`, `Date`, `Error`, `Map`, `Promise`, `RegExp`, `Set`, `Symbol`, and `URL`. Subtypes of `Error` are also supported as long as they have a globally available constructor on the client (`SyntaxError`, `TypeError`, etc.).

This may or may not require any immediate changes to your code once enabling Single Fetch:

- ✅ `json` responses returned from `loader`/`action` functions will still be serialized via `JSON.stringify` so if you return a `Date`, you'll receive a `string` from `useLoaderData`/`useActionData`
- ⚠️ If you're returning a `defer` instance or a naked object, it will now be serialized via `turbo-stream`, so if you return a `Date`, you'll receive a `Date` from `useLoaderData`/`useActionData`
  - If you wish to maintain current behavior (excluding streaming `defer` responses), you may just wrap any existing naked object returns in `json`

This also means that you no longer need to use the `defer` utility to send `Promise` instances over the wire! You can include a `Promise` anywhere in a naked object and pick it up on `useLoaderData().whatever`. You can also nest `Promise`'s if needed - but beware of potential UX implications.

Once adopting Single Fetch, it is recommended that you incrementally remove the usage of `json`/`defer` throughout your application in favor of returning raw objects.

### Streaming Timeout

Previously, Remix has a concept of an `ABORT_TIMEOUT` built-into the default [`entry.server.tsx`][entry-server] files which would terminate the React renderer, but it didn't do anything in particular to clean up any pending deferred promises.

Now that Remix is streaming internally, we can cancel the `turbo-stream` processing and automatically reject any pending promises and stream up those errors to the client. By default, this happens after 4950ms - a value that was chosen to be just under the current 5000ms `ABORT_DELAY` in most entry.server.tsx files - since we need to cancel the promises and let the rejections stream up through the React renderer prior to aborting the React side of things.

You can control this by exporting a `streamTimeout` numeric value from your `entry.server.tsx` and Remix will use that as the number of milliseconds after which to reject any outstanding Promises from `loader`/`action`'s. It's recommended to decouple this value from the timeout in which you abort the React renderer - and you should always set the React timeout to a higher value so it has time to stream down the underlying rejections from your `streamTimeout`.

```tsx filename=app/entry.server.tsx lines=[1-2,32-33]
// Reject all pending promises from handler functions after 5 seconds
export const streamTimeout = 5000;

// ...

function handleBrowserRequest(
  request: Request,
  responseStatusCode: number,
  responseHeaders: Headers,
  remixContext: EntryContext
) {
  return new Promise((resolve, reject) => {
    const { pipe, abort } = renderToPipeableStream(
      <RemixServer
        context={remixContext}
        url={request.url}
        abortDelay={ABORT_DELAY}
      />,
      {
        onShellReady() {
          /* ... */
        },
        onShellError(error: unknown) {
          /* ... */
        },
        onError(error: unknown) {
          /* ... */
        },
      }
    );

    // Automatically timeout the react renderer after 10 seconds
    setTimeout(abort, 10000);
  });
}
```

### Revalidations

Previously, Remix would always revalidate all active loaders after _any_ action submission, regardless of the result of the action. You could opt-out of revalidation on a per-route basis via [`shouldRevalidate`][should-revalidate].

With Single Fetch, if an `action` returns or throws a `Response` with a `4xx/5xx` status code, Remix will _not revalidate_ loaders by default. If an `action` returns or throws anything that is not a 4xx/5xx Response, then the revalidation behavior is unchanged. The reasoning here is that in most cases, if you return a `4xx`/`5xx` Response, you didn't actually mutate any data so there is no need to reload data.

If you _want_ to continue revalidating one or more loaders after a 4xx/5xx action response, you can opt-into revalidation on a per-route basis by returning `true` from your [`shouldRevalidate`][should-revalidate] function. There is also a new `actionStatus` parameter passed to the function that you can use if you need to decide based on the action status code.

Revalidation is handled via a `?_routes` query string parameter on the single fetch HTTP call which limits the loaders being called. This means that when you are doing fine-grained revalidation, you will have cache enumerations based on the routes being requested - but all of the information is in the URL so you should not need any special CDN configurations (as opposed to if this was done via a custom header that required your CDN to respect the `Vary` header).

[future-flags]: ../file-conventions/remix-config#future
[should-revalidate]: ../route/should-revalidate
[entry-server]: ../file-conventions/entry.server
[client-loader]: ../route/client-loader
[2.9.0]: https://github.com/remix-run/remix/blob/main/CHANGELOG.md#v290
[rfc]: https://github.com/remix-run/remix/discussions/7640
[turbo-stream]: https://github.com/jacob-ebey/turbo-stream
[rendertopipeablestream]: https://react.dev/reference/react-dom/server/renderToPipeableStream
[rendertoreadablestream]: https://react.dev/reference/react-dom/server/renderToReadableStream
[rendertostring]: https://react.dev/reference/react-dom/server/renderToString
[hydrateroot]: https://react.dev/reference/react-dom/client/hydrateRoot
[starttransition]: https://react.dev/reference/react/startTransition
[headers]: ../route/headers
[mdn-headers]: https://developer.mozilla.org/en-US/docs/Web/API/Headers
[resource-routes]: ../guides/resource-routes
[returning-response]: ../route/loader.md#returning-response-instances
[responsestub]: #headers
[streaming-format]: #streaming-data-format
[undici-polyfill]: https://github.com/remix-run/remix/blob/main/CHANGELOG.md#undici
[undici]: https://github.com/nodejs/undici
[csp]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy/script-src
[csp-nonce]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy/Sources#sources
[remix-utils]: https://github.com/sergiodxa/remix-utils
[merging-remix-and-rr]: https://remix.run/blog/merging-remix-and-react-router
[migration-guide]: #migrating-a-route-with-single-fetch
[breaking-changes]: #breaking-changes
[action-revalidation]: #streaming-data-format
[start]: #enabling-single-fetch
[type-inference-section]: #type-inference
[compatibility-flag]: https://developers.cloudflare.com/workers/configuration/compatibility-dates
