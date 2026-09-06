# Live data — real-time updates without polling

A dashboard does not have to poll. The platform already pushes data-change notifications, and
an embedded page can subscribe to them using the **platform's own mechanism** — captured from
the app bundle, not invented.

## What it is

A **Socket.IO** connection to `<api>/subscription`, authenticated by the viewer's token,
subscribing to per-model data-change binding keys. On create/update/delete the server pushes a
message carrying `{ body: { documentId, operation, … } }`.

## The binding key

This is the platform's own `toDataChangeNotificationRoutingKey`:

```
mfgx.tenant.<tenantId>.dataChangeNotification.application.<lowerFirst(Model)>.<operation>
```

`lowerFirst` only lowercases the first character — `WorkOrder` becomes `workOrder`, not
`workorder`. Get that wrong and you subscribe successfully to nothing.

> **The broker rejects wildcard bindings.** You cannot bind `….workOrder.#` and catch all three
> operations. Subscribe **once per concrete operation** — `create`, `update`, `delete` — for every
> model you care about.

## Loading the client

Fetch the server's own socket.io client and inject it as an inline `<script>` with
`textContent`:

```js
fetch(apiBase + '/subscription/socket.io.js')
  .then(function (r) { return r.text(); })
  .then(function (src) {
    var s = document.createElement('script');
    s.textContent = src;            // textContent, NOT eval — no unsafe-eval needed in CSP
    document.head.appendChild(s);
  });
```

Two reasons this beats bundling a client: it is **version-matched** to the server, and it is a
cache hit. Using `textContent` rather than `eval` keeps the page off `unsafe-eval`.

## What to do with a notification

Follow the platform's own `performLiveReload`: **flash the affected node, row or panel, then
re-fetch that one record by id.** Do not re-run the whole dashboard query — a change
notification carries the `documentId`, which is all you need for a targeted fetch, and a full
refresh throws away scroll position, selection and any 3D camera state.

## When polling is still right

Live subscription costs a socket per viewer. For a wallboard that nobody interacts with and
that tolerates a 30-second lag, a timed re-render is simpler and cheaper. Use subscriptions
when the viewer is *working* in the screen and needs to see their own write land.
