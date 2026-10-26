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

## Examples

Taken from this package's own tests, which compile and run on every release.

```osy
using Osysharp.Dns;
using Osysharp.Dns.Cloudflare;
using DnsCloudflareKitTests;
using Osysharp.Http;

[Test(Testers)]
void a_record_lands_in_the_longest_zone_that_ends_its_name_unproxied_with_the_token() {
  runas (T) {
    Cloudflare().SetRecord("billing--prod.eu.osyrin.test", DnsRecordKind.CNAME, "gw.zone-b.example.net.", 60);
    Cloudflare().SetRecord("shop--prod.osyrin.test", DnsRecordKind.A, "203.0.113.7", 30);
    var eu = FakeCfRecord.Single(r => r.Name == "billing--prod.eu.osyrin.test");
    Assert.Equal("zone-eu", eu.ZoneId);
    Assert.Equal("CNAME", eu.Type);
    Assert.Equal("gw.zone-b.example.net", eu.Content);
    Assert.False(eu.Proxied);
    var root = FakeCfRecord.Single(r => r.Name == "shop--prod.osyrin.test");
    Assert.Equal("zone-root", root.ZoneId);
    // Cloudflare's floor is 60 seconds; a shorter TTL is raised to it, not refused.
    Assert.Equal(60, root.Ttl);
    // Every call carried the token — and nothing else could have authenticated it.
    foreach (var call in FakeCfCall.ToList()) { Assert.Equal("Bearer cf-test-token", call.Authorization); }
  }
}
```

From [`tests/cloudflare.test.osy`](tests/cloudflare.test.osy).

```osy
using Osysharp.Dns;
using Osysharp.Dns.Cloudflare;
using DnsCloudflareKitTests;
using Osysharp.Http;

[Test(Testers)]
void an_address_that_keeps_its_kind_is_updated_in_place_and_one_that_changes_kind_replaces_the_other() {
  runas (T) {
    Standing("zone-root", "CNAME", "app--prod.osyrin.test", "gw.zone-a.example.net");
    Cloudflare().SetRecord("app--prod.osyrin.test", DnsRecordKind.CNAME, "gw.zone-b.example.net", 60);
    var moved = FakeCfRecord.Where(r => r.Name == "app--prod.osyrin.test").ToList();
    Assert.Equal(1, moved.Count);
    Assert.Equal("gw.zone-b.example.net", moved[0].Content);
    // Kept its kind: one PUT, no delete — the name never resolved to nothing.
    Assert.Equal(1, FakeCfCall.Where(c => c.Method == "PUT").ToList().Count);
    Assert.Equal(0, FakeCfCall.Where(c => c.Method == "DELETE").ToList().Count);

    // Changing kind: the alias goes and the address arrives — a CNAME cannot share its name.
    Cloudflare().SetRecord("app--prod.osyrin.test", DnsRecordKind.A, "203.0.113.9", 60);
    var after = FakeCfRecord.Where(r => r.Name == "app--prod.osyrin.test").ToList();
    Assert.Equal(1, after.Count);
    Assert.Equal("A", after[0].Type);
    Assert.Equal("203.0.113.9", after[0].Content);
  }
}
```

From [`tests/cloudflare.test.osy`](tests/cloudflare.test.osy).

```osy
using Osysharp.Dns;
using Osysharp.Dns.Cloudflare;
using DnsCloudflareKitTests;
using Osysharp.Http;

[Test(Testers)]
void writing_what_is_already_there_makes_no_write_at_all() {
  runas (T) {
    Standing("zone-root", "CNAME", "same--prod.osyrin.test", "gw.zone-a.example.net");
    var row = FakeCfRecord.Single(r => r.Name == "same--prod.osyrin.test");
    row.Ttl = 60;
    Cloudflare().SetRecord("same--prod.osyrin.test", DnsRecordKind.CNAME, "gw.zone-a.example.net", 60);
    Assert.Equal(0, FakeCfCall.Where(c => c.Method != "GET").ToList().Count);
  }
}
```

From [`tests/cloudflare.test.osy`](tests/cloudflare.test.osy).

## Learn more

- [Osysharp.Dns.Cloudflare in the package catalogue](https://osysharp.com/packages/#osysharp-dns-cloudflare)
- [The Osy# reference](https://osysharp.com/reference/) — the language, the runtime and every kit
- [`llms-full.txt`](https://osysharp.com/llms-full.txt) — the whole reference and specification as one Markdown file, for a coding agent

## Licence

MIT — see [`LICENSE`](LICENSE).
