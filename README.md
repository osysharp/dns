# Osysharp.Dns

DNS records an app writes, through whichever company hosts the zone. You get one record shape, a provider interface,
an app setting naming the provider, and a provider that keeps records in the app's own table. Use that last one for
tests, for a local machine, and to read what a real provider would be asked for.

```osy
var dns = App.Setting<DnsSetup>().Provider;
dns.SetRecord("billing--prod.example.com", DnsRecordKind.CNAME, "gw.eu-west.example.net", 60);
dns.AddTxtValue("_acme-challenge.example.com", proof, 60);      // ADDS a value; never replaces one
```

## Use it

```osy
// app.osy
use Osysharp.Dns@0;
use Osysharp.Dns.Cloudflare@0 { egress "api.cloudflare.com"; }   // the provider, or your own IDnsProvider
```

```osy
app.Dns = new DnsSetup {
  Provider = new CloudflareDns { ApiToken = Security.IsSecretSet("CloudflareDnsToken") ? Secret.CloudflareDnsToken : "" },
};
```

## The contract

`IDnsProvider` has five verbs plus `Key`, `IsReady()` and an optional `NotReadyReason()`:

| verb | what it does |
|---|---|
| `ListRecords(domain)` | every A, AAAA, CNAME and TXT record at the domain and under it |
| `SetRecord(name, kind, value, ttl)` | the name carries exactly ONE address record (A, AAAA or CNAME), replacing any other. Writing what is already there changes nothing. A TXT is refused. |
| `DeleteRecords(name)` | removes the name's address records. Its TXT values stay. |
| `AddTxtValue(name, value, ttl)` | **adds** one TXT value and leaves the others |
| `RemoveTxtValue(name, value)` | removes **one** TXT value and leaves the others |

⛔ **There is no TXT setter.** Two parties proving the same name publish two values at once — two zones answering ACME
DNS-01 for one wildcard, for example. A provider whose "set" replaced the name's TXT records would make the second
party erase the first.

`KindToPointAt(target)` picks the kind for you: an IPv4 address gives an A, an IPv6 address an AAAA, and a host name a
CNAME.

## The local provider

`LocalDnsProvider` keeps its records in `LocalDnsRecord`, in your app's own database, and publishes nothing. It obeys
the contract exactly, so a feature that is right against it is right against a real provider. The entity is
`Extensible`: you say who may reach it.

```osy
partial entity LocalDnsRecord { security { allow read, create, update, delete when IsOperator; } }
```

⚠ It is always ready. Never fall back to it because a real provider's credential is missing: a platform that did would
report every record as written while the internet resolved none of them.

## Providers

- **[Osysharp.Dns.Cloudflare](https://osyrin.com/templates/kits/dns-cloudflare/)**: Cloudflare's v4 API.
- **Your own**: Route 53, Google Cloud DNS, Azure DNS or a self-hosted server is one more provider package with the same
  five verbs.

## Examples

Taken from this package's own tests, which compile and run on every release.

```osy
using Osysharp.Dns;
using DnsKitTests;

[Test(Operators)]
void an_address_replaces_whatever_address_the_name_had_and_writing_it_again_changes_nothing() {
  runas (Op) {
    Dns().SetRecord("Billing--Prod.Example.test.", DnsRecordKind.CNAME, "gw.zone-a.example.test", 60);
    Dns().SetRecord("billing--prod.example.test", DnsRecordKind.CNAME, "gw.zone-b.example.test", 60);
    Dns().SetRecord("billing--prod.example.test", DnsRecordKind.CNAME, "gw.zone-b.example.test", 60);
    var rows = LocalDnsRecord.Where(r => r.Name == "billing--prod.example.test").ToList();
    Assert.Equal(1, rows.Count);
    Assert.Equal(DnsRecordKind.CNAME, rows[0].Kind);
    Assert.Equal("gw.zone-b.example.test", rows[0].Value);
    // An address replaces an alias too: a CNAME cannot share its name with anything.
    Dns().SetRecord("billing--prod.example.test", DnsRecordKind.A, "203.0.113.7", 60);
    var after = LocalDnsRecord.Single(r => r.Name == "billing--prod.example.test");
    Assert.Equal(DnsRecordKind.A, after.Kind);
    Assert.Equal("203.0.113.7", after.Value);
  }
}
```

From [`tests/local.test.osy`](tests/local.test.osy).

```osy
using Osysharp.Dns;
using DnsKitTests;

[Test(Operators)]
void a_txt_value_is_added_beside_the_values_already_there_and_never_replaces_one() {
  runas (Op) {
    Dns().AddTxtValue("_acme-challenge.example.test", "zone-a-proof", 60);
    Dns().AddTxtValue("_acme-challenge.example.test", "zone-b-proof", 60);
    Dns().AddTxtValue("_acme-challenge.example.test", "zone-b-proof", 60);
    var values = TxtValuesAt(Dns().ListRecords("example.test"), "_acme-challenge.example.test");
    Assert.Equal(2, values.Count);
    Assert.Contains("zone-a-proof", values);
    Assert.Contains("zone-b-proof", values);
    // Removing one leaves the other — zone A finishing its challenge must not end zone B's.
    Dns().RemoveTxtValue("_acme-challenge.example.test", "zone-a-proof");
    var left = TxtValuesAt(Dns().ListRecords("example.test"), "_acme-challenge.example.test");
    Assert.Equal(1, left.Count);
    Assert.Equal("zone-b-proof", left[0]);
    // Removing what is not there is not an error.
    Dns().RemoveTxtValue("_acme-challenge.example.test", "never-published");
  }
}
```

From [`tests/local.test.osy`](tests/local.test.osy).

```osy
using Osysharp.Dns;
using DnsKitTests;

[Test(Operators)]
void a_txt_value_cannot_be_written_with_the_address_setter() {
  runas (Op) {
    Assert.Throws(() => Dns().SetRecord("_acme-challenge.example.test", DnsRecordKind.TXT, "x", 60));
    Assert.Equal(0, LocalDnsRecord.Where(r => r.Kind == DnsRecordKind.TXT).ToList().Count);
  }
}
```

From [`tests/local.test.osy`](tests/local.test.osy).

## Learn more

- [Osysharp.Dns in the package catalogue](https://osysharp.com/packages/#osysharp-dns)
- [The Osy# reference](https://osysharp.com/reference/) — the language, the runtime and every kit
- [`llms-full.txt`](https://osysharp.com/llms-full.txt) — the whole reference and specification as one Markdown file, for a coding agent

## Licence

MIT — see [`LICENSE`](LICENSE).
