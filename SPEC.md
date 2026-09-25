# Quipu Metrics Protocol, version 1

Status: draft
Last updated: 2026-08-28

The release of this document is in `VERSION`. See the README for how that
number relates to the wire version below.

The key words MUST, MUST NOT, SHOULD, SHOULD NOT and MAY are to be interpreted
as described in RFC 2119.

---

## 1. Overview

A client is a library embedded in a Minecraft plugin, mod, or server. Every 30
minutes it collects a small set of facts about the host it runs on and sends
them to the backend as a single HTTP request. The backend stores the raw
submission and later aggregates it into public charts.

There is no authentication, no session, and no response payload the client acts
on. A submission is fire and forget.

---

## 2. Endpoint

```
POST https://api.quipumetrics.org/v1/data/{platform}
```

`{platform}` is a platform key from section 9. It partitions the public
statistics: a Bukkit server and a Velocity proxy are never counted in the same
population.

### 2.1 Request headers

| Header | Value | Required |
|--------|-------|----------|
| `Content-Type` | `application/json` | yes |
| `Content-Encoding` | `gzip` | yes |
| `Accept` | `application/json` | yes |
| `User-Agent` | `Quipu/1` | yes |

The body MUST be gzip compressed. The backend rejects uncompressed bodies.

### 2.2 Transport

- TLS is mandatory. A client MUST NOT fall back to plaintext HTTP.
- Connect timeout SHOULD be 5 seconds, read timeout 10 seconds.
- The request MUST NOT run on a thread that blocks gameplay. On platforms with
  a main game thread, only the collection of values may happen there; the HTTP
  call must be asynchronous.

### 2.3 Response codes

| Code | Meaning | Client action |
|------|---------|---------------|
| 202 | Accepted | none |
| 400 | Malformed payload | log if logging enabled, do not retry |
| 413 | Payload too large | log if logging enabled, do not retry |
| 429 | Rate limited | do not retry, wait for the next scheduled cycle |
| 5xx | Backend problem | do not retry |

The response body is informational only. A client MUST NOT parse it to decide
behaviour.

---

## 3. Retry policy

**A client MUST NOT retry a failed submission.**

On any failure the client discards that cycle and waits for the next scheduled
one. There is no queue, no backoff loop, and no buffering of missed
submissions.

This is not a simplification, it is a safety requirement. A backend outage
would otherwise be followed by every client in the world retrying at once,
turning a recovery into a second outage. Losing a data point is harmless: the
next one arrives within 30 minutes.

---

## 4. Timing

- After startup, a client MUST wait a random delay uniformly distributed
  between 3 and 6 minutes before its first submission.
- After the first submission, it MUST submit every 30 minutes.
- A client MUST NOT align submissions to wall clock boundaries such as `:00`
  and `:30`.

The random initial delay spreads load. Aligning to the clock would concentrate
every server on the planet into two spikes per hour.

---

## 5. Request body

```json
{
  "serverUUID": "3f2b1c4d-5e6f-4a7b-8c9d-0e1f2a3b4c5d",
  "playerAmount": 12,
  "onlineMode": 1,
  "platformVersion": "git-Paper-196 (MC: 1.21.1)",
  "javaVersion": "21.0.4",
  "osName": "Linux",
  "osArch": "amd64",
  "osVersion": "6.1.0-18-amd64",
  "coreCount": 8,
  "service": {
    "id": 1234,
    "version": "1.0.0",
    "charts": []
  }
}
```

### 5.1 Common fields

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `serverUUID` | string | yes | See section 6 |
| `playerAmount` | integer | yes | Players online at collection time. `-1` if unknown |
| `onlineMode` | integer | yes | `1` online mode, `0` offline mode, `-1` unknown |
| `platformVersion` | string | yes | Raw version string from the platform, unparsed |
| `javaVersion` | string | no | Omit on non-JVM clients |
| `osName` | string | yes | |
| `osArch` | string | yes | |
| `osVersion` | string | yes | |
| `coreCount` | integer | yes | Logical cores available to the process |

`platformVersion` MUST be sent exactly as the platform reports it. Parsing it
into server software and Minecraft version is the backend's job. A client that
pre-parses it destroys information the backend needs.

### 5.2 Fields that are never sent

A client MUST NOT send:

- IP addresses, hostnames, or domain names
- Player names, UUIDs, or any per player data
- File paths, world names, or server names
- Plugin or mod lists belonging to other authors

Geographic location is derived by the backend from the connection's source
address and stored only as an ISO 3166-1 alpha-2 country code. The address
itself is never persisted.

### 5.3 Service object

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `id` | integer | yes | Service id issued by quipumetrics.org |
| `version` | string | yes | Version of the plugin or mod |
| `charts` | array | yes | May be empty |

One submission carries exactly one service. A host running eight plugins sends
eight independent submissions that share one `serverUUID`.

### 5.4 One service id spans every platform

A project registers **one** service id and uses it from every platform it ships
on. The Bukkit build, the Velocity build and the Fabric build of the same
project all send the same `service.id`.

The platform is never encoded in the id. It comes from the request path, so the
backend can always split a service's data by platform without the author
registering anything extra.

This means:

- An author registers once, not once per platform.
- A project has one page showing its total reach, with a platform breakdown
  derived automatically.
- A chart that only makes sense on some platforms is simply omitted by the
  others, per section 7. It then shows data from the platforms that send it.

Two ids are for two products, not for two builds of one product.

---

## 6. Server UUID

The `serverUUID` identifies a physical host, not a plugin.

### 6.1 Generation

Generated once, as a random version 4 UUID. It MUST NOT be derived from the IP
address, hostname, MAC address, or any other machine property.

### 6.2 Persistence

The UUID MUST be stored in a configuration file shared by every Quipu client on
that host, at the path given in section 8. A client MUST read an existing value
rather than generating its own.

### 6.3 Why this matters

The backend counts distinct hosts by counting distinct `serverUUID` values. A
host with eight plugins sends eight submissions per cycle. If those eight
submissions carry eight different UUIDs, that host is counted as eight servers
and its players are counted eight times.

There is no way to repair this after the fact. An implementation that writes
the UUID into its own plugin folder instead of the shared location produces
data that is silently and permanently wrong.

---

## 7. Charts

Each entry in `service.charts`:

```json
{ "chartId": "language_used", "data": { "value": "es" } }
```

`chartId` MUST match `^[a-z0-9_]{1,64}$` and MUST correspond to a chart defined
for that service on quipumetrics.org. Unknown chart ids are discarded silently.

### 7.1 Data shapes

| Type | `data` |
|------|--------|
| `single_line` | `{"value": <integer>}` |
| `simple_pie` | `{"value": <string>}` |
| `advanced_pie` | `{"values": {<string>: <integer>}}` |
| `drilldown_pie` | `{"values": {<string>: {<string>: <integer>}}}` |
| `simple_bar` | `{"values": {<string>: <integer>}}` |
| `advanced_bar` | `{"values": {<string>: [<integer>, ...]}}` |
| `multi_line` | `{"values": {<string>: <integer>}}` |

A chart whose value cannot be determined MUST be omitted from the array. It
MUST NOT be sent with a null or placeholder value.

### 7.2 Limits

| Limit | Value |
|-------|-------|
| Charts per submission | 32 |
| Keys per chart | 64 |
| Chart label length | 128 characters |
| Uncompressed body size | 64 KiB |

Exceeding a limit results in `400` or `413`. The client SHOULD enforce these
locally rather than sending a request that will be rejected.

---

## 8. Configuration file

Format is Java `.properties`: `key=value`, one per line, `#` comments. Chosen
because every target language parses it without a dependency.

### 8.1 Location

| Platform family | Path, relative to server root |
|-----------------|-------------------------------|
| Bukkit, Sponge, BungeeCord | `plugins/Quipu/config.properties` |
| Velocity | `plugins/quipu/config.properties` |
| Fabric, Forge, NeoForge | `config/quipu/config.properties` |
| Everything else | `quipu/config.properties` |

### 8.2 Keys

| Key | Type | Default | Meaning |
|-----|------|---------|---------|
| `enabled` | boolean | `true` | Master switch for this host |
| `serverUuid` | string | generated | See section 6 |
| `logErrors` | boolean | `false` | Log failed submissions |
| `logSentData` | boolean | `false` | Log the payload before sending |

### 8.3 Opt out

When `enabled` is `false` the client MUST NOT open a connection to the backend
at all. Sending a submission that merely flags the host as opted out is not
acceptable.

The client MUST re-read this key on every cycle, so that an operator disabling
collection takes effect without a restart.

### 8.4 First run

If the file does not exist, the client creates it with the defaults and a
freshly generated `serverUuid`. Concurrent creation by several plugins starting
at once MUST be handled: create via a temporary file and an atomic rename, and
if the rename fails because the file now exists, read that file instead.

---

## 9. Platform keys

| Key | Family | Platform |
|-----|--------|----------|
| `bukkit` | `server` | Bukkit, Spigot, Paper, Purpur, Folia and forks |
| `sponge` | `server` | SpongeVanilla, SpongeForge |
| `minestom` | `server` | Minestom |
| `nukkit` | `server` | Nukkit, PowerNukkitX, Cloudburst |
| `bungeecord` | `proxy` | BungeeCord, Waterfall and forks |
| `velocity` | `proxy` | Velocity |
| `fabric` | `mod` | Fabric, Quilt |
| `forge` | `mod` | Minecraft Forge |
| `neoforge` | `mod` | NeoForge |
| `other` | `other` | Anything without an assigned key |

A key MUST match `^[a-z0-9_]{1,32}$`. New keys are added by opening a pull
request against this repository. Until a key is assigned, use `other`.

### 9.1 Families

A family groups keys for display. The backend derives it from the key using
this table. A client never sends a family, so a client cannot misreport one.

| Family | Meaning |
|--------|---------|
| `server` | Runs the world and holds the main game thread |
| `proxy` | Routes players between servers, has no world |
| `mod` | Loaded by a mod loader alongside other mods |
| `other` | Unassigned, including platforms outside the JVM |

Families exist so the built in platform chart can be a `drilldown_pie`: the
outer ring is the family, the inner ring is the key. A service shipping on nine
platforms shows four readable slices instead of nine thin ones, and the detail
is still one click away.

Adding a key never requires a client change. Assign its family here and every
existing service picks up the grouping.

### 9.2 Implementations

A platform key names an API, not the software serving it. `bukkit` covers
CraftBukkit, Spigot, Paper, Purpur, Folia and every fork, which are very
different things to a server operator deciding what to run.

The backend therefore derives a third level, the implementation, by parsing
`platformVersion`.

| Level | Example | Source |
|-------|---------|--------|
| Family | `server` | Derived from the key, section 9.1 |
| Key | `bukkit` | Request path |
| Implementation | `Paper` | Parsed from `platformVersion` |

Parsing rules:

- The client MUST NOT send the implementation. It sends `platformVersion` raw,
  per section 5.1, and the backend does the parsing. A client that guessed its
  own implementation could report a name no other client uses, fragmenting the
  chart.
- When `platformVersion` carries no implementation, the implementation is the
  key's display name. A Fabric client reporting `1.21.4` is `Fabric`.
- When parsing fails, the implementation MUST be `Unknown`. A submission is
  never discarded because its version string was not recognised.
- Parsing rules live in the backend and may change at any time. Because raw
  strings are stored, a rule fix re-derives history correctly.

Known limit: a fork that reports itself as its upstream is counted as that
upstream. Nothing in a version string can reveal a fork that chose to hide.

Because the implementation is a third level and a `drilldown_pie` holds two,
a service gets two built in charts rather than one three level chart:

| Chart | Outer ring | Inner ring |
|-------|------------|------------|
| Platforms | Family | Key |
| Server software | Key | Implementation |

Both charts count servers, and the same split applies to player totals, so an
author can see that their plugin runs on 400 Paper hosts and 60 Purpur hosts
rather than 460 undifferentiated `bukkit` ones.

### 9.3 The platform chart counts pairs, not hosts

The unit of the built in platform chart is the distinct pair of `serverUUID`
and platform key, not the host.

A host running a Velocity proxy and a Paper backend, both carrying the same
service, is one host but counts once under `proxy` and once under `server`.
Slices therefore add up to more than the number of hosts reported elsewhere,
and this is correct: the question the chart answers is which platforms a
service runs on, not how many machines exist.

Any label shown to users MUST NOT call this figure a host count. See section
10.1.

The same applies to the server software chart of section 9.2.

---

## 10. Backend obligations

Stated here because clients depend on them.

- The backend MUST accept a submission without authentication.
- The backend MUST NOT persist the source IP address. It may resolve it to a
  country code at ingest time and MUST discard it afterwards.
- The backend MUST rate limit per `serverUUID` and per source address, and MUST
  answer `429` rather than dropping silently, so client authors can detect the
  condition while testing.
- The backend MUST NOT discard a submission merely because another submission
  with the same `serverUUID` and `service.id` already arrived in the same cycle.
  See section 10.1.

### 10.1 Several instances on one host

One physical host commonly runs several server instances at once: a proxy and
its backends, or a lobby and a survival server, all under the same operating
system.

Those instances share one configuration file, therefore one `serverUUID`. When
the same plugin runs on several of them, that host legitimately sends several
submissions per cycle carrying an identical `serverUUID` and an identical
`service.id`. They may also carry an identical platform key, because two
Paper instances on one machine are both `bukkit`.

No combination of the fields in this specification distinguishes those
instances, and none is added, because nothing downstream needs to tell them
apart.

The consequences are binding:

- Deduplicating by `serverUUID` plus `service.id` is forbidden. It would
  silently delete real data from every multi instance host.
- Rate limits MUST be applied per `serverUUID` across all services, with a
  ceiling high enough for a host running many instances and many plugins. A
  limit of one submission per service per cycle is wrong.
- The `servers` figure a service reports is the count of distinct `serverUUID`
  values, which is a count of **hosts**, not of server instances. Any label
  shown to users MUST say hosts.

When the backend runs behind a reverse proxy or a tunnel, the proxy MUST
forward the original source address, and the backend MUST trust that header
only from its own proxy. Without this, every submission appears to originate
from the proxy and geographic statistics collapse to a single country.

---

## 11. Reference payloads

See `examples/`. Every file there validates against
`schema/submission.schema.json` and forms part of the backend's test suite.
