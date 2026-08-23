# JSON Feeder for PassWall2

A small LuCI converter embedded in **Services → PassWall2 → JSON Feeder**. It accepts pasted JSON or a local `.json` file and writes supported proxies into PassWall2's `nodes` sections.

## Supported JSON shapes

- Xray: `{ "outbounds": [...] }`
- sing-box: `{ "outbounds": [...] }` and `{ "endpoints": [...] }` (sing-box 1.11+ keeps WireGuard there)
- Clash JSON: `{ "proxies": [...] }`
- PassWall2-shaped data: `{ "nodes": [...] }`, a node array, or one node object
- Share links carried inside the JSON: a bare string item, a `link`/`url`/`uri` field on an object, or `{ "links": [...] }`

Supported protocols are VMess, VLESS, Trojan, Shadowsocks, SOCKS, HTTP, Hysteria2, AnyTLS, and WireGuard. Non-proxy outbounds such as `direct`, `block`, `dns`, and selectors are skipped.

### WireGuard

WireGuard is read from every shape above and written to PassWall2's `wireguard_*` options: the peer endpoint becomes the node `address`/`port`, and private key, peer public key, pre-shared key, local addresses, MTU, keep-alive, and reserved bytes are carried over. `reserved` given as a JSON array is converted to PassWall2's decimal form (`78,251,145`); a Base64 string is kept as-is.

`wireguard://` (also `wg://`) links are accepted when they appear inside the JSON, for example:

```json
{
  "links": [
    "wireguard://<private-key>@engage.cloudflareclient.com:2408?publickey=<peer-key>&address=172.16.0.2/32,2606:4700:110::2/128&mtu=1280&reserved=78,251,145#WARP"
  ]
}
```

A link with no `#name` fragment inherits the `remarks`/`name`/`tag` of the object that carries it. The input is still parsed as JSON — a raw link pasted on its own is not valid JSON and is rejected by the browser before it is sent.

Each successful export:

1. validates and converts the entire input before changing UCI;
2. replaces only nodes marked as created by JSON Feeder;
3. commits the nodes to `/etc/config/passwall2`;
4. saves the source JSON at `/etc/passwall2-json-feeder.json` with mode `0600`.

The **Clear saved JSON** button removes only the saved source JSON. It intentionally leaves exported nodes in place.

## Build

Place `luci-app-passwall2-json-feeder` under `package/` (or an OpenWrt feed), then run:

```sh
make menuconfig
# LuCI -> Applications -> luci-app-passwall2-json-feeder
make package/luci-app-passwall2-json-feeder/compile V=s
```

Install the resulting package and refresh LuCI's cache:

```sh
opkg install luci-app-passwall2-json-feeder_1.1.0-1_all.ipk
rm -rf /tmp/luci-indexcache /tmp/luci-modulecache
/etc/init.d/uhttpd restart
```

For OpenWrt 25.12 and newer, install the generated package with:

```sh
apk add --allow-untrusted luci-app-passwall2-json-feeder-1.1.0-r1.apk
```

## Notes

- PassWall2 is a required dependency.
- The input limit is 2 MiB.
- The converter imports the first server/user from each Xray outbound, matching PassWall2's one-node-per-section model.
- Core selection prefers the source core when installed and otherwise falls back to an installed Xray or sing-box core.
- WireGuard needs a core built with WireGuard support: Xray, or sing-box with the `with_wireguard` tag.

## Tests

The converter can be exercised without a router. From the package directory:

```sh
lua5.4 scripts/test-converter.lua
```
