# Universal Dhan Market Data Engine v1.1

One DhanHQ v2 data gateway for multiple independent PWAs. Strategies stay in each PWA; the backend ingests, normalizes, caches and distributes market data.

## Included
- DhanHQ v2 WebSocket feed with binary packet decoding for ticker, quote, OI, previous-close and full packets.
- Shared market cache and dynamic WebSocket subscriptions for multiple PWA clients.
- REST fallbacks for spot/quote and historical intraday candles.
- Server-side candle aggregation for 3-minute candles from 1-minute history.
- Option-chain expiry list and chain snapshot routes with a cache respecting Dhan's 3-second unique-request limit.
- Option-chain fields normalized to LTP, OI, change-in-OI, volume, IV, Delta, Gamma, Theta, Vega, bid/ask and quantities.
- Web Push VAPID public-key endpoint, subscription registration and test push endpoint.
- Health, status, config, ticks, state and subscription endpoints.
- Render configuration and copy/paste environment block.

## Deploy on Render
1. Extract this ZIP and create a GitHub repository from its contents, or upload the folder to your repository.
2. Create a Render Web Service, runtime Node, build `npm install`, start `npm start`.
3. Add your own `DHAN_CLIENT_ID`, six-digit `DHAN_PIN`, and the Base32 `DHAN_TOTP_SECRET` copied when setting up TOTP. The backend generates a fresh TOTP and obtains the 24-hour access token automatically, then refreshes it before expiry. You do not need to enter a daily access token. `DHAN_ACCESS_TOKEN` is an optional fallback only. Keep all secrets in Render Environment, never in the PWA or a public repository.
4. Health check: `/api/health`; broker state: `/api/status`. Status reports `authMode: "TOTP_AUTO"` when TOTP automation is configured..

## Common endpoints
- `GET /api/health`
- `GET /api/status`
- `GET /api/config`
- `GET /api/state`
- `GET /api/spot?symbol=NIFTY`
- `GET /api/ticks?symbol=NIFTY`
- `GET /api/candles?symbol=NIFTY&timeframe=1m&limit=250`
- `GET /api/quote?symbol=NIFTY`
- `GET /api/option-chain/expiries?symbol=NIFTY`
- `GET /api/option-chain?symbol=NIFTY&expiry=YYYY-MM-DD` (expiry may be omitted to use first active expiry)
- `POST /api/option-chain` JSON `{ "symbol":"NIFTY", "expiry":"YYYY-MM-DD" }`
- `GET /api/notifications/vapid-public-key`
- `POST /api/push/subscribe` JSON `{ "clientId":"my-pwa", "subscription": <PushSubscription JSON> }`
- `POST /api/push/test` JSON `{ "title":"Test", "body":"Push test" }`
- `WS /ws`

## WebSocket protocol
Connect to `wss://YOUR-HOST/ws`. Send:
```json
{"type":"subscribe","symbols":["NIFTY"],"streams":["ticks","candles"],"timeframe":"1m"}
```
Server events include `hello`, `status`, `subscribed`, and `tick`. For known option contracts, send a symbol/security id plus `exchangeSegment` to `POST /api/subscribe`; the browser WebSocket subscription message is optimized for configured index aliases. The backend shares one upstream Dhan feed and fans data out to clients.

## Universal data contract
The backend deliberately does not run PWA strategy logic. `/api/candles` returns `{symbol,timeframe,candles:[{time,open,high,low,close,volume,openInterest,source}]}`. `/api/spot` returns price, previous close, change, timestamp, source and stale flags. Option-chain fields are normalized into `rows[]` with `strike`, `ce`, `pe` and Greek/market fields.

## Important constraints
- Dhan Client ID, PIN and TOTP secret must be valid, and the account must have the required API/data entitlements. Access tokens expire after 24 hours; this version regenerates them automatically using TOTP. Dhan rate limits, allowed WebSocket connections and instrument limits still apply.
- The backend cannot manufacture missing Greeks/depth or infer actual dealer intent. It exposes broker fields when available and labels its source.
- In-memory caches and push subscriptions reset when the service restarts; use persistent storage before using this as a multi-user commercial service.
- `CORS_ORIGIN=*` is convenient for initial testing, not a production security boundary. Restrict origins and add authentication/rate limiting before public multi-client use.
- The service is data/notification only; it does not place orders.
