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
- **Not built yet**: Route 53, Google Cloud DNS, Azure DNS, and a self-hosted server. Each is one more kit with the same
  five verbs.
