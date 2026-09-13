# sun-moon-terminal-kds

The **KDS (Kitchen Display System)**: shows accepted orders and reports
조리완료.

React + TypeScript, no server of its own — it builds to static files
served from the `:8000` hub at `/kds/`. Same shape as
[sun-moon-terminal-pos](https://github.com/schware/sun-moon-terminal-pos),
one screen earlier in the order's life.

## How it fits

```
Order(8083) ──ACCEPTED event──▶ Device Server(8087) ──WebSocket──▶ this screen
                                     ▲                                  │
                                     └──── GET /orders, POST /orders/status ┘
```

The Device Server decides who sees what — `ACCEPTED` goes to KDS,
`PLACED` to POS, `PRODUCED` to DID
(`OrderEventSubscriber.audienceFor`). This screen never learns where
Order lives.

**The WebSocket is a nudge, not the data** — see
sun-moon-terminal-pos's README for the reasoning. It also polls every
15 seconds so a missed nudge cannot leave a kitchen staring at an empty
screen.

## Build

```bash
npm install
npm run build          # → dist/
```

| Variable | Default |
|---|---|
| `VITE_DEVICE_SERVER` | `http://211.217.183.232:8087` |

Build-time, not runtime — see sun-moon-terminal-pos's README for why
the default is the public address rather than the LAN one.

## Deploy

```bash
tar -czf - -C dist . | ssh schware@192.168.0.2 \
  'rm -rf ~/apps/docker/nginx/html/kds/assets && tar xzf - -C ~/apps/docker/nginx/html/kds'
```

No restart — nginx serves that directory from a volume mount. **Check in
a private window** — `index.html` is not content-hashed and nginx sends
no `Cache-Control` for it, so a stale bundle can keep serving after a
deploy. sun-moon-terminal-pos's README has the full story and the log
command to confirm which bundle a browser actually took.

## Store and device id

Identical scheme to sun-moon-terminal-pos: per-tab `sessionStorage`, a
`localStorage` prefill only, `?storeId=&deviceId=` pinning a screen bolted
to a counter. See that repo's README for why.

## Not built yet

- **A second KDS screen in the same kitchen finds out late.** The Device
  Server only pushes `ACCEPTED` events to KDS
  (`OrderEventSubscriber.audienceFor`) — there is no push for "an order
  left the KDS queue." A screen that marks 조리완료 itself removes that
  order immediately (the click's own `reload()`), but a second screen in
  the same store only notices via its own 15s poll. Confirmed live on
  2026-09-13: moving an order to `PRODUCED` from outside the UI left it
  showing on an already-open KDS tab for up to 15s before the poll caught
  up. Fixing it for real means Device Server pushing something on
  `PRODUCED` to KDS too, not just to DID.
- **No authentication**, for the same reason as sun-moon-terminal-pos:
  every order in this system is a sample by design.
- **One item per order.** The kitchen ticket shows `menuName` as a single
  line because an order is one menu item today — no cart yet.
- Device ids are not checked against BO's device list — same open item
  as POS's `AcceptKnownFormatDirectory`.
