# flyaffinity

Session-affinity routing for [Fly.io](https://fly.io) apps that embed a
[parley](https://github.com/richardwooding/parley) relay: pin every connection
for a session to one machine so each machine's in-memory relay stays
authoritative for its shard, and you can run more than one.

parley keeps session state as live socket ownership (not serializable), so you
scale by sharding, not shared storage. This package supplies the
platform-specific half that parley deliberately leaves out via
`relay.Options.Router`:

- **Ownership by rendezvous (HRW) hashing** over the live machine set — moves
  only ~1/N sessions when the fleet changes (vs ~all for modulo), with no
  coordination (a fixed seedless hash, so every machine agrees).
- **Discovery from Fly internal DNS** (`vms.<app>.internal` TXT), cached briefly.
- **`Route`** plugs into `relay.Options.Router`: serve here when off-Fly,
  single-machine, already-replayed (one-hop `Fly-Replay-Src` loop guard), or
  this machine is the owner; otherwise reply `Fly-Replay: instance=<owner>`.

```go
aff := flyaffinity.New(os.Getenv("FLY_APP_NAME"), os.Getenv("FLY_MACHINE_ID"), 7*time.Second)
relaySrv := relay.New(relay.Options{Router: aff.Route})
// aff.Peers(ctx) feeds a dashboard.NewAggregator for cluster-wide stats.
```

Off Fly (`FLY_MACHINE_ID` empty) or at one machine, `Route` always serves
here — identical to a single-node relay. Requires clients that send parley's
`?s=<session-hex>` routing hint (parley's `session` package does this since
v0.5.0). Server-only (imports `parley/relay`); not part of a WASM build.

Extracted from [kibitz](https://github.com/richardwooding/kibitz) /
[confab](https://github.com/richardwooding/confab), which both run on it.

MIT licensed.
