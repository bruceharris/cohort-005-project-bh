# React Router 8 live updates: what is supported

Question (issue #4): what can React Router 8.3.0 (framework mode, `@react-router/serve` on Node) use to get a new Notification onto a page that is already open? This note lists facts only. It does not make a decision.

Sources: the published npm tarballs `react-router@8.3.0`, `@react-router/serve@8.3.0`, `@react-router/express@8.3.0`, `@react-router/node@8.3.0`, `compression@1.8.1` (I read their `dist` code), the React Router docs at reactrouter.com, and MDN.

## 1. Loader revalidation on navigation and mutation (built in, no extra code)

- By default React Router re-runs the loaders of matched routes after every action submission (`<Form>`, `useFetcher().submit`). It also re-runs them when URL search params change. It does not do this for every navigation: on a plain link navigation it re-runs only loaders whose params changed. A route can override this with the `shouldRevalidate` export. Docs: https://reactrouter.com/start/framework/route-module#shouldrevalidate , https://reactrouter.com/explanation/state-management
- What this means for notifications: a root or layout loader that returns the unread count gets fresh data whenever the user submits anything, and on navigations where that loader re-runs. A page that sits idle never updates.
- Cost: in framework mode, revalidation is a single-fetch request (`*.data`) that runs all the revalidating loaders on the server in one round trip.

## 2. Polling with `useRevalidator` (built in, timer is up to you)

- `useRevalidator()` returns `{ revalidate, state }`. In 8.3.0 the source is just `router.revalidate()` (react-router `dist/production/index.js`, `function useRevalidator`). Docs: https://reactrouter.com/api/hooks/useRevalidator
- Calling `revalidate()` re-runs every active loader, not only the notification loader. Each loader's `shouldRevalidate` still applies, so you can opt routes out. One poll therefore costs one HTTP request, plus whatever SQLite queries all the active loaders make.
- No polling or "revalidate on window focus" behaviour is built in. I grepped the 8.3.0 dist for `visibilitychange` and `focus` and found no such feature. You write the `setInterval` (or the visibility listener) yourself in a `useEffect`. The docs show this pattern.
- An alternative is `useFetcher().load('/some/resource-route')` on an interval. This polls one small resource route without revalidating the whole page.
- Latency is at most the poll interval. Load is (open tabs × 1/interval) requests per second.

## 3. Server-Sent Events from a resource route (works with @react-router/serve)

- A resource route (a route module with a `loader` and no default component) can return a `Response` whose body is a `ReadableStream`, with `Content-Type: text/event-stream`. Docs: https://reactrouter.com/how-to/resource-routes
- What `@react-router/serve` 8.3.0 does with that response (source):
  - The server is Express 5 with `compression()` middleware (`@react-router/serve/dist/cli.js`, line 113: `if (!isRSCBuild) app.use(compression())`).
  - `@react-router/express` calls `res.flushHeaders()` when the Content-Type matches `text/event-stream`. It also aborts `request.signal` when the client disconnects (`res.on("close", () => controller.abort())`). Use that signal to clear intervals and unsubscribe listeners.
  - `@react-router/node`'s `writeReadableStreamToWritable` calls `res.flush()` after every chunk if `flush` exists, which it does when compression is active. So in 8.3.0, compression does not hold back SSE events. Sending `Cache-Control: no-transform` still makes compression skip the response entirely (`compression/index.js`, line 296), which is the safer setting.
- Client side: use `new EventSource(url)` in a `useEffect`. The browser reconnects automatically. The server can set `retry:` and `id:`, and the browser sends `Last-Event-ID` on reconnect. MDN: https://developer.mozilla.org/en-US/docs/Web/API/EventSource , https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
- Connection limit: MDN says that without HTTP/2, the browser allows only 6 SSE connections per browser and domain, shared across all tabs. A 7th tab blocks. Node's plain `http` server (which `serve` uses) is HTTP/1.1 only. Over HTTP/2, which a TLS proxy in front may provide, the limit is negotiated (100 by default).
- Behind proxies: buffering proxies such as nginx (`proxy_buffering`) hold events back unless you send `X-Accel-Buffering: no` or disable buffering. Idle timeouts (for example nginx `proxy_read_timeout`, 60s by default) cut quiet streams, so send a `: ping` comment every ~15–30s. Docs: https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_buffering
- Fan-out: there is no pub/sub here. better-sqlite3 has no change notifications. To push, the server needs an in-process `EventEmitter` that the code creating a Notification calls. Alternatively, each SSE connection polls SQLite on the server. An in-process emitter only works with a single Node process; multiple instances would need shared pub/sub.
- Each open connection holds a socket and its closures for its whole lifetime. The loader runs once when the connection opens, so auth (cookie session) can be done there.
- SSE only carries data. After an event arrives, the page still has to call `revalidate()` (or update local state) to re-render.

## 4. Other options

- WebSockets: `@react-router/serve` exposes no hook for HTTP upgrade. You would need a custom Express server (`@react-router/express` `createRequestHandler`) to attach `ws`.
- React Router has nothing else built in for push.

## Summary table

| Approach | Built in? | Latency | Server cost | Gotchas |
|---|---|---|---|---|
| Revalidate on navigation or action | yes | only when the user acts | none extra | an idle page never updates |
| `useRevalidator` + interval | hook yes, timer no | up to the poll interval | 1 request per tab per tick; re-runs all loaders | use `shouldRevalidate` to narrow; pause when hidden |
| `useFetcher().load` polling | yes | up to the poll interval | 1 small request | page loaderData is not refreshed |
| SSE resource route | works out of the box | immediate | 1 long-lived socket per tab | 6-connection HTTP/1.1 limit, proxy buffering and timeouts, single-process fan-out |
