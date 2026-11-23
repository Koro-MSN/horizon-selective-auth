# horizon-selective-auth

A patch for [Horizon](https://github.com/RVZO6/horizon) (the Rust
reimplementation of Minecraft: Java Edition) that adds **selective session
authentication** to a dedicated server running in offline mode.

## The problem

With `online-mode=false`, a server never verifies who is connecting: the
player's UUID is derived from the name they claim, and the whitelist is, in
effect, a list of names. Anyone who learns a whitelisted name can connect
under it — taking over that player's identity, inventory, and (if the name
has operator status) the server itself.

Turning `online-mode=true` fixes that, but it also locks out Bedrock
players who join through Geyser/Floodgate, and it changes every Java
player's UUID to their online UUID, orphaning their player data.

## What this patch does

Two new `server.properties` keys, used with `online-mode=false`:

| Key | Default | Meaning |
| --- | --- | --- |
| `auth-selective` | `false` | Verify direct connections with the Mojang session service; exempt Floodgate-proxied (Bedrock) connections. |
| `auth-enforce` | `true` | Refuse sessions that fail verification. When `false`, failures are logged as warnings and the player is admitted anyway. |

With `auth-selective=true`:

- **Direct (Java) connections** go through the normal online-mode
  authentication: encryption challenge, then a `hasJoined` check against
  the session service. Only a client signed in to the account that owns
  the name can pass.
- **Floodgate connections** (detected by the NUL-separated data Geyser
  appends to the handshake host string) skip authentication — those
  players were already authenticated on the Xbox side.
- **Identity does not change.** A verified player keeps their offline
  UUID and the offline profile path, so player data, whitelist entries,
  and any existing skin handling are untouched. The session check is used
  purely as proof that the connection owns the name.
- Vanilla `online-mode=true` behaviour is unchanged.

## Rollout

1. Apply the patch and set `auth-selective=true`, `auth-enforce=false`.
2. Watch the log: sessions that would be refused are logged as
   `WARN ... (admitted anyway: auth-enforce is false)`. Have every regular
   player join once and confirm nobody legitimate shows up there.
3. Set `auth-enforce=true` and restart.

Note: `auth-enforce=false` only forgives a *completed* handshake whose
session is invalid or unverifiable. A client that cannot perform the
encrypted handshake at all is disconnected in both modes — such a client
could never pass enforcement either.

## Limitations

- Floodgate detection is the NUL separator in the handshake host string.
  If you front the server with a proxy that uses the same convention
  (e.g. legacy BungeeCord forwarding), those forwarded connections are
  exempt too — which matches the usual "the proxy authenticated them"
  model, but do not enable this patch if that is not your setup.
- Verified players do **not** adopt their Mojang profile properties;
  skins keep coming from whatever the offline path already does.

## Applying

The patch is against upstream commit `7fca3c2f` of `RVZO6/horizon`:

```sh
git clone https://github.com/RVZO6/horizon.git
cd horizon
git apply path/to/patches/0001-selective-auth.patch
```

It touches four files: `crates/horizon-client/src/lan/conn.rs` (the login
branch and the authentication outcome), `crates/horizon-client/src/lan/mod.rs`
(the network config), `crates/horizon-client/src/game/dedicated.rs`
(keypair generation and startup logging), and
`crates/horizon-server-files/src/properties.rs` (the two new keys).

## Status

Verified to type-check against the base commit with
`cargo check -p horizon-client` and `cargo check -p horizon-server-files`.
Not yet run-tested against a live server; the `auth-enforce=false` rollout
above is there so the first live test cannot lock anyone out.

## License

MIT, the same license as Horizon itself. This patch is a derivative work
of the Horizon project; all credit for the original code belongs to its
authors.
