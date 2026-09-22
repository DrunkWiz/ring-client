# ring-client

A typed, dependency-free Python client for the **Ring Partner API**, with an
offline emulator built from real captured responses — so you can develop and
test a Ring integration with no hardware, no subscription, and no 30-minute
token expiring in the middle of a debugging session.

Ring ships no SDK in any language. Every integrator writes the same `urllib`
wrapper and rediscovers the same handful of surprises. This is that wrapper,
with the surprises written down in the tests.

MIT licensed. Extracted from [Threshold](https://github.com/DrunkWiz/threshold-ring),
a doorbell that explains itself.

## Installing

There is nothing to install. Python 3.11 or newer, **zero runtime
dependencies** — copy the `ring_client/` folder into your project, or vendor
it as a submodule.

## Using it

Generate a token at the [Ring Playground](https://developer.amazon.com/ring/console/playground).
It lasts 30 minutes.

```python
from ring_client import RingClient, token_scopes, token_expiry

client = RingClient(token)

print(token_scopes(token))     # ('ava.v1:read',) — read-only, and nothing says so
print(token_expiry(token))     # checked locally, no round-trip

for device in client.devices():
    print(device.id, device.name)
    print(client.capabilities(device.id).max_resolution)
    print(client.status(device.id).online)
```

Live video is WHEP. The browser owns the peer connection, so your server
passes the SDP offer through and keeps the token out of the page:

```python
answer, session_url = client.start_whep_session(device_id, sdp_offer)
```

### Without any hardware or token

The emulator answers the same calls from payloads actually captured from the
Playground — including the empty event history and the null sensors, because
a fake that only returns the happy path teaches you nothing.

```python
from ring_client import RingClient, Emulator, DEVICE_ID

client = RingClient("any.token.value", opener=Emulator().opener)
client.devices()               # the captured Playground device
```

The transport is a plain callable, so the emulator drops straight into tests.

## What it covers

`devices` · `capabilities` · `configurations` · `status` · `user` ·
`locations` · `history` · `start_whep_session`

Every response is a typed dataclass over the JSON:API shape, so you are not
indexing into nested dictionaries by hand.

## Errors

One class per failure the API actually produces, rather than a relayed status
code: `TokenExpired`, `TokenInvalid`, `Forbidden`, `NotFound`, `RateLimited`,
`Unreachable`, all under `RingError`.

Token expiry is detected locally from the JWT before a request is made, so an
expired token tells you it expired instead of returning a puzzling 403.

## What the Ring API taught us the hard way

Written down so the next person does not lose the same hours:

- The base URL is `api.amazonvision.com`, not a `ring.com` host.
- Event history lives at `/v1/history/devices/{id}/events`.
  `/v1/devices/{id}/events` returns **403, not 404** — which sends you hunting
  for a permissions problem when you have made a typo.
- Playground tokens are **read-only** and nothing on the page says so. Decode
  the JWT to see the scopes.
- The Playground gives you a working **virtual device** — no hardware, no
  Protection plan — even though the development guide and the FAQ both say
  testing requires real devices.
- **Simulated events never reach event history**, so the event-driven path
  cannot be exercised end to end.
- CORS blocks browser calls from anywhere but the console, so `Failed to
  fetch` tells you nothing about whether an endpoint exists.

## Tests

```bash
python3 scripts/run_tests.py     # py on Windows
```

No network: the runner replaces `socket.connect`, so any outbound connection
fails loudly rather than silently passing. Every test runs against the
emulator.

## Licence

MIT.
