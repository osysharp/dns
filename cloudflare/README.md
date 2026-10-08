# Osysharp.Dns.Cloudflare

Cloudflare DNS as an [Osysharp.Dns](https://osyrin.com/templates/kits/dns/) provider. It is written in Osy# over `Http.*`, and it can reach
only `api.cloudflare.com`.

```osy
use Osysharp.Dns@0;
use Osysharp.Dns.Cloudflare@0 { egress "api.cloudflare.com"; }

app.Dns = new DnsSetup {
  Provider = new CloudflareDns { ApiToken = Security.IsSecretSet("CloudflareDnsToken") ? Secret.CloudflareDnsToken : "" },
};
```

## The token

Create it under **My Profile → API Tokens** with two permissions, **Zone · DNS · Edit** and **Zone · Zone · Read**.
Scope it to the zones it writes and no others. The app names its own secret, and this package never learns which one
it is. Without a token the provider is not ready, and `NotReadyReason()` says which permissions the token needs.

## What it does

- **It finds the zone itself.** It lists the zones the token can see and writes into the one with the longest name
  that ends the record's name. If no zone matches, the refusal names the zones the token *can* see.
- **It proxies only tunnel records.** Every record is "DNS only", so a name resolves to the address it names and
  Cloudflare's TLS does not sit in front of it. The one exception is a CNAME to a Cloudflare Tunnel
  (`<id>.cfargotunnel.com`). It is written proxied, because Cloudflare sends a name to a tunnel only through its edge.
- **An address keeps its kind in place.** A CNAME that changes target is one `PUT`, so the name never resolves to
  nothing in between. A change of kind is a delete and then a create, because Cloudflare refuses a CNAME beside
  anything else.
- **Adding a TXT value creates one more record.** Values that already stand are never read and rewritten. Removing a
  value deletes exactly the record holding it.
- **The TTL floor is 60 seconds.** A shorter TTL is raised to 60 rather than refused.
- **A refusal carries every error Cloudflare returned.** For a 401 or 403 it also names the permissions the token
  needs.

The tests run against a fake Cloudflare that keeps its records in a table (`tests/`). No test reaches the real API.
